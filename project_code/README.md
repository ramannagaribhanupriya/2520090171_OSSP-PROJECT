/*
 * Linux Alarm Clock  -  OSSP Project (Section 13, Team 17)
 *
 * OS concepts used:
 *   - POSIX threads  : background monitor thread (pthread_create / mutex)
 *   - Timers/Signals : setitimer(ITIMER_REAL) -> SIGALRM drives the ringing
 *   - File I/O       : open/read/write/close for persistent alarm storage
 *   - nanosleep()    : sleeps between checks (no busy-waiting)
 *
 * Build : make          Run : ./alarm_clock
 */
#define _POSIX_C_SOURCE 200809L
#include <errno.h>
#include <fcntl.h>
#include <pthread.h>
#include <semaphore.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/time.h>
#include <time.h>
#include <unistd.h>

#define MAX_ALARMS   32
#define LABEL_LEN    64
#define ALARM_FILE   "alarms.txt"
#define RING_TIMES   10          /* how many beeps per alarm            */

typedef struct {
    int    hour, minute;
    char   label[LABEL_LEN];
    long   last_fired;           /* minute-stamp of last trigger        */
} Alarm;

static Alarm           alarms[MAX_ALARMS];
static int             alarm_count = 0;
static pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
static volatile sig_atomic_t running   = 1;
static volatile sig_atomic_t ring_left = 0;
static volatile sig_atomic_t silenced = 0;  /* set by 'Stop ringing' */
static int   is_wsl = 0;                    /* running under WSL?    */
static sem_t beep_sem;                      /* handler -> beeper thread */
static char ring_msg[128];       /* message printed by the SIGALRM handler */
static size_t ring_len = 0;

/* ===================== FILE HANDLING (persistence) ===================== */

static void save_alarms(void)           /* caller must hold lock */
{
    int fd = open(ALARM_FILE, O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) { perror("open"); return; }
    char line[128];
    for (int i = 0; i < alarm_count; i++) {
        int n = snprintf(line, sizeof line, "%02d:%02d %s\n",
                         alarms[i].hour, alarms[i].minute, alarms[i].label);
        if (write(fd, line, n) < 0) { perror("write"); break; }
    }
    close(fd);
}

static void load_alarms(void)
{
    int fd = open(ALARM_FILE, O_RDONLY);
    if (fd < 0) return;                          /* no file yet: fine */
    char buf[8192];
    ssize_t n = read(fd, buf, sizeof buf - 1);
    close(fd);
    if (n <= 0) return;
    buf[n] = '\0';

    pthread_mutex_lock(&lock);
    alarm_count = 0;
    for (char *line = strtok(buf, "\n"); line && alarm_count < MAX_ALARMS;
         line = strtok(NULL, "\n")) {
        int h, m, off = 0;
        if (sscanf(line, "%d:%d %n", &h, &m, &off) >= 2 &&
            h >= 0 && h < 24 && m >= 0 && m < 60) {
            Alarm *a = &alarms[alarm_count++];
            a->hour = h; a->minute = m; a->last_fired = -1;
            snprintf(a->label, LABEL_LEN, "%s", line + off);
        }
    }
    pthread_mutex_unlock(&lock);
}

/* ===================== SIGNAL HANDLING (SIGALRM) ======================= */

/* Only async-signal-safe calls here: write(), setitimer(). */
static void on_sigalrm(int sig)
{
    (void)sig;
    if (ring_left > 0) {
        ssize_t r = write(STDOUT_FILENO, ring_msg, ring_len);
        (void)r;
        sem_post(&beep_sem);            /* wake the beeper thread */
        ring_left--;
    }
    if (ring_left <= 0) {                        /* stop the periodic timer */
        struct itimerval off = {{0, 0}, {0, 0}};
        setitimer(ITIMER_REAL, &off, NULL);
    }
}

/* Start ringing: arm a 1-second repeating timer; each tick raises SIGALRM */
static void start_ringing(const Alarm *a)
{
    ring_len = (size_t)snprintf(ring_msg, sizeof ring_msg,
        "\a\n*** ALARM!  %02d:%02d  %s ***\n", a->hour, a->minute, a->label);
    silenced  = 0;
    ring_left = RING_TIMES;
    struct itimerval t = {{1, 0}, {0, 1000}};    /* interval 1s, first in 1ms */
    setitimer(ITIMER_REAL, &t, NULL);
}

/* ===================== BEEP SOUND THREAD =============================== */

/* Play one real beep. Tries the best available method for this system. */
static void play_beep(void)
{
    if (is_wsl) {                         /* Windows speakers via PowerShell */
        if (system("powershell.exe -NoProfile -Command "
                   "\"[console]::Beep(1000,500)\" >/dev/null 2>&1") == 0) return;
    }
    if (system("paplay /usr/share/sounds/freedesktop/stereo/alarm-clock-elapsed.oga"
               " >/dev/null 2>&1") == 0) return;
    if (system("aplay -q /usr/share/sounds/alsa/Front_Center.wav"
               " >/dev/null 2>&1") == 0) return;
    if (system("beep >/dev/null 2>&1") == 0) return;
    ssize_t r = write(STDOUT_FILENO, "\a", 1);   /* last resort: terminal bell */
    (void)r;
}

/* Waits on a semaphore posted by the SIGALRM handler, then beeps.
 * (system() is not signal-safe, so it must not run inside the handler.)  */
static void *beeper_thread(void *arg)
{
    (void)arg;
    for (;;) {
        if (sem_wait(&beep_sem) == -1 && errno == EINTR) continue;
        if (!running) break;
        if (!silenced) play_beep();
    }
    return NULL;
}

/* ===================== BACKGROUND MONITOR THREAD ======================= */

static void *monitor_thread(void *arg)
{
    (void)arg;
    while (running) {
        time_t now = time(NULL);
        struct tm tm;
        localtime_r(&now, &tm);
        long stamp = (long)(now / 60);

        pthread_mutex_lock(&lock);
        for (int i = 0; i < alarm_count; i++) {
            Alarm *a = &alarms[i];
            if (a->hour == tm.tm_hour && a->minute == tm.tm_min &&
                a->last_fired != stamp) {
                a->last_fired = stamp;
                start_ringing(a);
            }
        }
        pthread_mutex_unlock(&lock);

        /* nanosleep: no busy-waiting. Resume if interrupted by SIGALRM. */
        struct timespec req = {1, 0}, rem;
        while (nanosleep(&req, &rem) == -1 && errno == EINTR && running)
            req = rem;
    }
    return NULL;
}

/* ===================== SYSTEM TIME-ZONE SYNC =========================== */

/* Turn an offset like "+05:30" into a POSIX TZ string ("LOCAL-5:30").
 * POSIX TZ uses the opposite sign: east of UTC is negative.             */
static int offset_to_tz(const char *txt, char *out, size_t outlen)
{
    char sign; int h, m;
    if (sscanf(txt, " %c%d:%d", &sign, &h, &m) != 3) return -1;
    if ((sign != '+' && sign != '-') || h < 0 || h > 14 || m < 0 || m > 59)
        return -1;
    snprintf(out, outlen, "LOCAL%c%d:%02d", sign == '+' ? '-' : '+', h, m);
    return 0;
}

/* Under WSL, Linux defaults to UTC while Windows shows local time.
 * Ask Windows for its real offset so the alarm clock matches the system. */
static void sync_system_timezone(void)
{
    FILE *f = fopen("/proc/version", "r");
    if (!f) return;
    char ver[512] = "";
    if (!fgets(ver, sizeof ver, f)) ver[0] = '\0';
    fclose(f);
    is_wsl = (strstr(ver, "icrosoft") != NULL || strstr(ver, "WSL") != NULL);
    if (!is_wsl) return;                  /* native Linux: nothing to sync */

    FILE *p = popen("powershell.exe -NoProfile -Command "
                    "\"(Get-Date).ToString('zzz')\" 2>/dev/null", "r");
    if (!p) return;
    char line[64] = "", tz[32];
    if (fgets(line, sizeof line, p) && offset_to_tz(line, tz, sizeof tz) == 0) {
        setenv("TZ", tz, 1);
        tzset();
    }
    pclose(p);
}

/* ===================== COMMAND-LINE INTERFACE ========================== */

static void add_alarm(void)
{
    char in[128], label[LABEL_LEN] = "Alarm";
    int h, m;
    printf("Enter time (HH:MM, 24-hour): ");
    if (!fgets(in, sizeof in, stdin)) return;
    if (sscanf(in, "%d:%d", &h, &m) != 2 || h < 0 || h > 23 || m < 0 || m > 59) {
        printf("Invalid time.\n"); return;
    }
    printf("Label (optional): ");
    if (fgets(in, sizeof in, stdin)) {
        in[strcspn(in, "\n")] = '\0';
        if (in[0]) { strncpy(label, in, LABEL_LEN - 1); label[LABEL_LEN - 1] = '\0'; }
    }
    pthread_mutex_lock(&lock);
    if (alarm_count >= MAX_ALARMS) {
        pthread_mutex_unlock(&lock);
        printf("Alarm list full.\n"); return;
    }
    Alarm *a = &alarms[alarm_count++];
    a->hour = h; a->minute = m; a->last_fired = -1;
    snprintf(a->label, LABEL_LEN, "%s", label);
    save_alarms();
    pthread_mutex_unlock(&lock);
    printf("Alarm set for %02d:%02d (%s)\n", h, m, label);
}

static void list_alarms(void)
{
    pthread_mutex_lock(&lock);
    if (alarm_count == 0) printf("No alarms set.\n");
    for (int i = 0; i < alarm_count; i++)
        printf("  %d) %02d:%02d  %s\n", i + 1,
               alarms[i].hour, alarms[i].minute, alarms[i].label);
    pthread_mutex_unlock(&lock);
}

static void delete_alarm(void)
{
    char in[32];
    list_alarms();
    printf("Alarm number to delete: ");
    if (!fgets(in, sizeof in, stdin)) return;
    int n = atoi(in);
    pthread_mutex_lock(&lock);
    if (n < 1 || n > alarm_count) {
        pthread_mutex_unlock(&lock);
        printf("Invalid number.\n"); return;
    }
    for (int i = n - 1; i < alarm_count - 1; i++) alarms[i] = alarms[i + 1];
    alarm_count--;
    save_alarms();
    pthread_mutex_unlock(&lock);
    printf("Deleted.\n");
}

static void stop_ringing(void)
{
    ring_left = 0;
    silenced  = 1;
    struct itimerval off = {{0, 0}, {0, 0}};
    setitimer(ITIMER_REAL, &off, NULL);
    printf("Alarm silenced.\n");
}

static void show_time(void)
{
    time_t now = time(NULL);
    char buf[64];
    strftime(buf, sizeof buf, "%A, %d %b %Y  %H:%M:%S (UTC%z)", localtime(&now));
    printf("Current time: %s\n", buf);
}

static void menu(void)
{
    printf("\n===== LINUX ALARM CLOCK =====\n"
           "1. Set alarm\n2. List alarms\n3. Delete alarm\n"
           "4. Stop ringing\n5. Show current time\n6. Exit\n> ");
    fflush(stdout);
}

int main(void)
{
    struct sigaction sa;
    memset(&sa, 0, sizeof sa);
    sa.sa_handler = on_sigalrm;
    sa.sa_flags   = SA_RESTART;          /* keep fgets() working after signal */
    sigemptyset(&sa.sa_mask);
    sigaction(SIGALRM, &sa, NULL);

    sync_system_timezone();      /* make time match the Windows/system clock */
    load_alarms();

    sem_init(&beep_sem, 0, 0);
    pthread_t tid, btid;
    if (pthread_create(&tid, NULL, monitor_thread, NULL) != 0 ||
        pthread_create(&btid, NULL, beeper_thread, NULL) != 0) {
        perror("pthread_create"); return 1;
    }

    char in[32];
    for (;;) {
        menu();
        if (!fgets(in, sizeof in, stdin)) break;
        switch (atoi(in)) {
            case 1: add_alarm();    break;
            case 2: list_alarms();  break;
            case 3: delete_alarm(); break;
            case 4: stop_ringing(); break;
            case 5: show_time();    break;
            case 6: goto done;
            default: printf("Choose 1-6.\n");
        }
    }
done:
    running = 0;
    sem_post(&beep_sem);                 /* wake beeper so it can exit */
    pthread_join(tid, NULL);
    pthread_join(btid, NULL);
    printf("Goodbye!\n");
    return 0;
}
