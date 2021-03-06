# 4 時計

## 4.1

 * 時刻
   * 1970月1月1日0時〜
 * 時限処理
   * あとでやる
   * デバイスポーリング
   * SIGALRM
 * グローバルタイマ
   * do_timer
 * CPUローカルタイマ
   * smp_local_timer_interrupt

## グローバルタイマー割り込みハンドラ

### do_timer ロードアベレージの計算

CentOS6.5 だと書籍とだいぶ違う。 **TODO** x86 の割り込みからのエントリポイントは?

```c
/*
 * The 64-bit jiffies value is not atomic - you MUST NOT read it
 * without sampling the sequence number in xtime_lock.
 * jiffies is defined in the linker script...
 */
void do_timer(unsigned long ticks)
{
	jiffies_64 += ticks;
	update_wall_time();
	calc_global_load();
}
```

 * update_wall_time で xtime の更新
   * クロックソースに応じて更新される
 * calc_global_load でロードアベレージの計算をする

```c
/*
 * calc_load - update the avenrun load estimates 10 ticks after the
 * CPUs have updated calc_load_tasks.
 */
void calc_global_load(void)
{
	unsigned long upd = calc_load_update + 10;
	long active;

	if (time_before(jiffies, upd))
		return;

	active = atomic_long_read(&calc_load_tasks);
	active = active > 0 ? active * FIXED_1 : 0;

	avenrun[0] = calc_load(avenrun[0], EXP_1, active);
	avenrun[1] = calc_load(avenrun[1], EXP_5, active);
	avenrun[2] = calc_load(avenrun[2], EXP_15, active);

	calc_load_update += LOAD_FREQ;
}
```

移動平均?

```c
static unsigned long
calc_load(unsigned long load, unsigned long exp, unsigned long active)
{
	load *= exp;
	load += active * (FIXED_1 - exp);
	return load >> FSHIFT;
}
```

## 4.2.2 CPUローカルな時計 - ハードウェア割り込み

 * CPUローカルな割り込みはタイミングをコアごとにズラす
   * 競合を避ける
   * 負荷が集中する事をさける
 * プロファイリング
 * scheduler_tick

## smp_local_timer_interrupt (2.6.15)

CONFIG_SMP で update_process_times を呼び出すか否かが変わる

 * update_process_times
   * account_user_time
   * account_system_time
     * acct_update_integrals

```c
/*
 * Local timer interrupt handler. It does both profiling and
 * process statistics/rescheduling.
 *
 * We do profiling in every local tick, statistics/rescheduling
 * happen only every 'profiling multiplier' ticks. The default
 * multiplier is 1 and it can be changed by writing the new multiplier
 * value into /proc/profile.
 */

inline void smp_local_timer_interrupt(struct pt_regs * regs)
{
	int cpu = smp_processor_id();

	profile_tick(CPU_PROFILING, regs);

    // プロファイラを起動???
	if (--per_cpu(prof_counter, cpu) <= 0) {
		/*
		 * The multiplier may have changed since the last time we got
		 * to this point as a result of the user writing to
		 * /proc/profile. In this case we need to adjust the APIC
		 * timer accordingly.
		 *
		 * Interrupts are already masked off at this point.
		 */
		per_cpu(prof_counter, cpu) = per_cpu(prof_multiplier, cpu);
		if (per_cpu(prof_counter, cpu) !=
					per_cpu(prof_old_multiplier, cpu)) {
			__setup_APIC_LVTT(
					calibration_result/
					per_cpu(prof_counter, cpu));
			per_cpu(prof_old_multiplier, cpu) =
						per_cpu(prof_counter, cpu);
		}

#ifdef CONFIG_SMP
        // ((regs->xcs & 3) | (regs->eflags & VM_MASK)) != 0;
        // CSレジスタ ... ???
        // EFLAFGS    ... VM_MASK 仮想X86モード?
		update_process_times(user_mode_vm(regs));
#endif
	}

	/*
	 * We take the 'long' return path, and there every subsystem
	 * grabs the apropriate locks (kernel lock/ irq lock).
	 *
	 * we might want to decouple profiling from the 'long path',
	 * and do the profiling totally in assembly.
	 *
	 * Currently this isn't too much of an issue (performance wise),
	 * we can take more than 100K local irqs per second on a 100 MHz P5.
	 */
}
```

update_process_times でローカルタイマ割り込みを受けた際の task_struct の時間の統計を出す

 * run_local_timers (softirq の実行)
 * CPU時間の統計
 * 再スケジューリング
   * scheduler_tick
 * posix タイマ (struct k_timer) => timer_create(2)
   * check_thread_timers
     * RLIMIT_RTTIME
   * check_process_timers
     * CPUCLOCK_PROF
     * CPUCLOCK_VIRT
     * CPUCLOCK_SCHED
     * RLIMIT_CPU を超えてないかどうか

```c
/*
 * Called from the timer interrupt handler to charge one tick to the current 
 * process.  user_tick is 1 if the tick is user time, 0 for system.
 */
void update_process_times(int user_tick)
{
	struct task_struct *p = current;
	int cpu = smp_processor_id();

	/* Note: this timer irq context must be accounted for as well. */
	if (user_tick)
		account_user_time(p, jiffies_to_cputime(1));
	else
		account_system_time(p, HARDIRQ_OFFSET, jiffies_to_cputime(1));
	run_local_timers();
	if (rcu_pending(cpu))
		rcu_check_callbacks(cpu, user_tick);

    /* 再スケジューリング */
	scheduler_tick();

    /* posix タイマ */
 	run_posix_cpu_timers(p);
}
```

account_user_time, account_system_time でタスクのCPU統計と、CPUの統計を出す

 * account_system_time が IRQ, SOFTIRQ, system時間, ... と分類されていておもしろい
   * munin のグラフで見るのもこれ
 * account_user_time は nice 時間が特殊かな?

```c
/*
 * Account user cpu time to a process.
 * @p: the process that the cpu time gets accounted to
 * @hardirq_offset: the offset to subtract from hardirq_count()
 * @cputime: the cpu time spent in user space since the last update
 */
void account_user_time(struct task_struct *p, cputime_t cputime)
{
	struct cpu_usage_stat *cpustat = &kstat_this_cpu.cpustat;
	cputime64_t tmp;

    // プロセスの user 時間に加算
	p->utime = cputime_add(p->utime, cputime);

	/* Add user time to cpustat. */
	tmp = cputime_to_cputime64(cputime);

    // CPU自体がどう使われたのかの統計?
    // NICE 値が 0 以上なら nice 時間として加算
	if (TASK_NICE(p) > 0)
		cpustat->nice = cputime64_add(cpustat->nice, tmp);
	else
		cpustat->user = cputime64_add(cpustat->user, tmp);
}

/*
 * Account system cpu time to a process.
 * @p: the process that the cpu time gets accounted to
 * @hardirq_offset: the offset to subtract from hardirq_count()
 * @cputime: the cpu time spent in kernel space since the last update
 */
void account_system_time(struct task_struct *p, int hardirq_offset,
			 cputime_t cputime)
{
	struct cpu_usage_stat *cpustat = &kstat_this_cpu.cpustat;
	runqueue_t *rq = this_rq();
	cputime64_t tmp;

	p->stime = cputime_add(p->stime, cputime);

	/* Add system time to cpustat. */
	tmp = cputime_to_cputime64(cputime);

    // IRQ, SOFTIRQ, system時間, I/O wait、idle で分類している
	if (hardirq_count() - hardirq_offset)
		cpustat->irq = cputime64_add(cpustat->irq, tmp);
	else if (softirq_count())
		cpustat->softirq = cputime64_add(cpustat->softirq, tmp);
	else if (p != rq->idle)
		cpustat->system = cputime64_add(cpustat->system, tmp);
	else if (atomic_read(&rq->nr_iowait) > 0)
		cpustat->iowait = cputime64_add(cpustat->iowait, tmp);
	else
		cpustat->idle = cputime64_add(cpustat->idle, tmp);
	/* Account for system time used */
	acct_update_integrals(p);
}
```

acct_update_integrals で acct(2) アカウンティング統計を更新?

```c
/**
 * acct_update_integrals - update mm integral fields in task_struct
 * @tsk: task_struct for accounting
 */
void acct_update_integrals(struct task_struct *tsk)
{
	if (likely(tsk->mm)) {
		long delta = tsk->stime - tsk->acct_stimexpd;

		if (delta == 0)
			return;
		tsk->acct_stimexpd = tsk->stime;
		tsk->acct_rss_mem1 += delta * get_mm_rss(tsk->mm);
		tsk->acct_vm_mem1 += delta * tsk->mm->total_vm;
	}
}
```

### scheduler_tick

HZの頻度で呼び出しされる (= ローカルタイマ)

```
/*
 * This function gets called by the timer code, with HZ frequency.
 * We call it with interrupts disabled.
 *
 * It also gets called by the fork code, when changing the parent's
 * timeslices.
 */
void scheduler_tick(void)
{
	int cpu = smp_processor_id();
	struct rq *rq = cpu_rq(cpu);
	struct task_struct *curr = rq->curr;

	sched_clock_tick();

	spin_lock(&rq->lock);
	update_rq_clock(rq);
	update_cpu_load_active(rq);
	curr->sched_class->task_tick(rq, curr, 0);
	spin_unlock(&rq->lock);

	perf_event_task_tick();

#ifdef CONFIG_SMP
	rq->idle_at_tick = idle_cpu(cpu);
	trigger_load_balance(rq, cpu);
#endif
}
```

## 4.2.3 CPUローカルな時計 - ソフト割り込み

 * タイマーリスト (struct timer_list) のハンドラ実行
 * 縮退
   * 複数回のローカルタイマ割り込みに対して、一回しかローカルタイマソフト割り込みが実行されないこと
   * まとめてやりなおす

```c
/*
 * Called by the local, per-CPU timer interrupt on SMP.
 */
void run_local_timers(void)
{
    // hrtimer ? でキューイングされてるジョブを実行?
	hrtimer_run_queues();
	raise_softirq(TIMER_SOFTIRQ);
}
```

TIMER_SOFTIRQ に対応するのは run_timer_softirq。 init_timers で登録されている

```c

void __init init_timers(void)
{
	int err = timer_cpu_notify(&timers_nb, (unsigned long)CPU_UP_PREPARE,
				(void *)(long)smp_processor_id());

	init_timer_stats();

	BUG_ON(err == NOTIFY_BAD);
	register_cpu_notifier(&timers_nb);
	open_softirq(TIMER_SOFTIRQ, run_timer_softirq);
}
```

run_timer_softirq の実装

```c
/*
 * This function runs timers and the timer-tq in bottom half context.
 */
static void run_timer_softirq(struct softirq_action *h)
{
	struct tvec_base *base = __get_cpu_var(tvec_bases);

	hrtimer_run_pending();

	if (time_after_eq(jiffies, base->timer_jiffies))
		__run_timers(base);
}
```

__run_timers でタイマ (struct timer_list) の実行?

 * keepalive のタイマとか?
 * timer_list の実装サンプル [refs](https://github.com/hiboma/kernel_module_scratch/blob/master/tasklet/tasklet.c)

```c
/**
 * __run_timers - run all expired timers (if any) on this CPU.
 * @base: the timer vector to be processed.
 *
 * This function cascades all vectors and executes all expired timer
 * vectors.
 */
static inline void __run_timers(struct tvec_base *base)
{
	struct timer_list *timer;

	spin_lock_irq(&base->lock);
	while (time_after_eq(jiffies, base->timer_jiffies)) {
		struct list_head work_list;
		struct list_head *head = &work_list;
		int index = base->timer_jiffies & TVR_MASK;

		/*
		 * Cascade timers:
		 */
		if (!index &&
			(!cascade(base, &base->tv2, INDEX(0))) &&
				(!cascade(base, &base->tv3, INDEX(1))) &&
					!cascade(base, &base->tv4, INDEX(2)))
			cascade(base, &base->tv5, INDEX(3));
		++base->timer_jiffies;
		list_replace_init(base->tv1.vec + index, &work_list);
		while (!list_empty(head)) {
			void (*fn)(unsigned long);
			unsigned long data;

            /* タイマ */
			timer = list_first_entry(head, struct timer_list,entry);
			fn = timer->function;
			data = timer->data;

			timer_stats_account_timer(timer);

			set_running_timer(base, timer);
			detach_timer(timer, 1);

			spin_unlock_irq(&base->lock);
			{
				int preempt_count = preempt_count();

#ifdef CONFIG_LOCKDEP
				/*
				 * It is permissible to free the timer from
				 * inside the function that is called from
				 * it, this we need to take into account for
				 * lockdep too. To avoid bogus "held lock
				 * freed" warnings as well as problems when
				 * looking into timer->lockdep_map, make a
				 * copy and use that here.
				 */
				struct lockdep_map lockdep_map =
					timer->lockdep_map;
#endif
				/*
				 * Couple the lock chain with the lock chain at
				 * del_timer_sync() by acquiring the lock_map
				 * around the fn() call here and in
				 * del_timer_sync().
				 */
				lock_map_acquire(&lockdep_map);

				trace_timer_expire_entry(timer);

                /* タイマの実行 */
				fn(data);
				trace_timer_expire_exit(timer);

				lock_map_release(&lockdep_map);

				if (preempt_count != preempt_count()) {
					printk(KERN_ERR "huh, entered %p "
					       "with preempt_count %08x, exited"
					       " with %08x?\n",
					       fn, preempt_count,
					       preempt_count());
					BUG();
				}
			}
			spin_lock_irq(&base->lock);
		}
	}
	set_running_timer(base, NULL);
	spin_unlock_irq(&base->lock);
}
```

## 4.3 Linux の時刻

### 4.3.1 Linux 内部時計

 * UTC 1970/1/1 からの経過時間を **xtime**
 * stime(2), settimeofday(2)
 * NTP slew mode

### 4.3.2 ハードウェアのカレンダ

 * **カレンダ** と呼ばれるコントローラー
 * hwclock <=> Linux 内部時計
 * BIOS とかでセットするやつ
 * /dev/rtc

```
[vagrant@vagrant-centos65 ~]$ ls -hal /dev/rtc*
lrwxrwxrwx 1 root root      4 May 19 12:44 /dev/rtc -> rtc0
crw-rw---- 1 root root 254, 0 May 19 12:44 /dev/rtc0
```

### 4.3.4 タイムスタンプカウンタ

#### TSC Time Stamp Counter

 * [access.redhat.com 15.1. ハードウェアクロック](https://access.redhat.com/site/documentation/ja-JP/Red_Hat_Enterprise_MRG/2/html/Realtime_Reference_Guide/chap-Realtime_Reference_Guide-Timestamping.html)
   * クロックソース
 * http://msmania.wordpress.com/tag/rdtsc/
   * プロセッサごとに用意されてるレジスタ
   * 高速な理由

> それが、RDTSC (=Read Time Stamp Counter) 命令です。詳細は IA-32 の仕様書に書いてありますが、RDTSC を実行すると、1 命令で Tick 値を取得することができます。
> しかも単位はクロック単位です。Pentium 以降の IA-32 アーキテクチャーでは、プロセッサーごとに TSC (=Time Stamp Counter) という 64bit カウンターが MSR (マシン固有レジスタ) に含まれており、RDTSC はこれを EDX:EAX レジスターにロードする命令です。

/var/log/messages に TSC の同期を試みるログが出る

```
Jan 28 12:40:21 vagrant-centos65 kernel: TSC synchronization [CPU#0 -> CPU#1]:
Jan 28 12:40:21 vagrant-centos65 kernel: Measured 1826989 cycles TSC warp between CPUs, turning off TSC clock.
Jan 28 12:40:21 vagrant-centos65 kernel: Marking TSC unstable due to check_tsc_sync_source failed
```

## 4.4 各種タイマー関連ハードゥエア

## 4.5 時刻の取得

> 周期的に発生するタイマー割り込みの精度以上にはなりません。Intel x86用Linuxの場合、タイマ割り込み間隔は4ミリ秒です。

> ただし、最近のCPUは周波数を動的に変更できるものも増えてきたため、可能であれば、CPU内部のレジスタよりはCPU外部のハードウェアを利用するようになっています。

 * [インテルターボ・ブースト・テクノロジー](http://ja.wikipedia.org/wiki/%E3%82%A4%E3%83%B3%E3%83%86%E3%83%AB_%E3%82%BF%E3%83%BC%E3%83%9C%E3%83%BB%E3%83%96%E3%83%BC%E3%82%B9%E3%83%88%E3%83%BB%E3%83%86%E3%82%AF%E3%83%8E%E3%83%AD%E3%82%B8%E3%83%BC)
 * [Intel SpeedStep テクノロジ](http://ja.wikipedia.org/wiki/Intel_SpeedStep_%E3%83%86%E3%82%AF%E3%83%8E%E3%83%AD%E3%82%B8)

----

 * time(2), gettimeofday(2)
   * time は精度が秒
   * gettimeofday は精度が マイクロ秒

```c
#if 0
#!/bin/bash
CFLAGS="-O2 -std=gnu99 -lrt -W -Wall -fPIE -D_FORTIFY_SOURCE=2"
o=`basename $0`
o=".${o%.*}"
gcc ${CFLAGS} -o $o $0 && ./$o $*; exit
#endif

#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <sys/time.h>

int main()
{
	time_t t;
	if (time(&t) == -1) {
		perror("time");
		exit(1);
	}

	struct timeval tv;
	if (gettimeofday(&tv, NULL) == -1) {
		perror("gettimeofday");
		exit(1);
	}

	struct timespec ts;
	if (clock_gettime(CLOCK_REALTIME, &ts) == -1) {
		perror("clock_gettime");
		exit(1);
	}

	printf("%ld\n", t);
	printf("%ld - %ld\n", tv.tv_sec, tv.tv_usec);
	printf("%ld - %ld\n", ts.tv_sec, ts.tv_nsec);
	exit(0);

}
```

time_t, timeval, timespec の精度を確認しておこう

型　| 精度
---|---
time_t | 秒
struct timeval | マイクロ秒
struct timespec | ナノ秒

### time(2) の実装

**内部時計 = xtime** から時刻を取る実装になっている。非常にシンプル

```c
/*
 * sys_time() can be implemented in user-level using
 * sys_gettimeofday().  Is this for backwards compatibility?  If so,
 * why not move it into the appropriate arch directory (for those
 * architectures that need it).
 */
SYSCALL_DEFINE1(time, time_t __user *, tloc)
{
    // ->
	time_t i = get_seconds();

	if (tloc) {
		if (put_user(i,tloc))
			return -EFAULT;
	}
	force_successful_syscall_return();
	return i;
}
```

get_seconds の中身は **timekeeper.xtime.tv_sec** を return するだけ

```c
unsigned long get_seconds(void)
{
	return timekeeper.xtime.tv_sec;
}
EXPORT_SYMBOL(get_seconds);
```

timekeeper.xtime.tv_sec がどのように更新されているか? は update_wall_time を読むとよい

### gettimeofday(2) の実装

> gettimeofdayシステムコールでは、システムコールインターフェイス上は、ナノ秒単位の時刻を得ることかできます。

gettimeofday(2) は **struct timeval** なので、精度がマイクロ秒に丸まってないかな? 記述ミスかな

```c
SYSCALL_DEFINE2(gettimeofday, struct timeval __user *, tv,
		struct timezone __user *, tz)
{
	if (likely(tv != NULL)) {
		struct timeval ktv;
        // ->
		do_gettimeofday(&ktv);
		if (copy_to_user(tv, &ktv, sizeof(ktv)))
			return -EFAULT;
	}
	if (unlikely(tz != NULL)) {
		if (copy_to_user(tz, &sys_tz, sizeof(sys_tz)))
			return -EFAULT;
	}
	return 0;
}
```

do_gettimeofday の中身

  * tv_sec の扱いは time(2) と一緒
  * tv_usec は getnstimeofday の中身を見るといい
    * getnstimeofday で得た値を 1/1000 してマイクロ秒に丸めている

```c
/**
 * do_gettimeofday - Returns the time of day in a timeval
 * @tv:		pointer to the timeval to be set
 *
 * NOTE: Users should be converted to using getnstimeofday()
 */
void do_gettimeofday(struct timeval *tv)
{
	struct timespec now;

    // ->
	getnstimeofday(&now);
	tv->tv_sec = now.tv_sec;
    // nanosec を 1000 で割って usec に変換している
	tv->tv_usec = now.tv_nsec/1000;
}
EXPORT_SYMBOL(do_gettimeofday);
```

getnstimeofday(2) は ナノ秒を精度として時刻取得する

 * シーケンスロック * [refs](http://wiki.bit-hive.com/north/pg/%A5%B7%A5%B1%A1%BC%A5%F3%A5%B9%A5%ED%A5%C3%A5%AF)
   * 時計は書き手より読み手の方が圧倒的に多いはずなので、シーケンスロックを使っているのだろう

```c
/**
 * getnstimeofday - Returns the time of day in a timespec
 * @ts:		pointer to the timespec to be set
 *
 * Returns the time of day in a timespec.
 */
void getnstimeofday(struct timespec *ts)
{
	unsigned long seq;
	s64 nsecs;

	WARN_ON(timekeeping_suspended);    k

	do {
        // シーケンスロック
		seq = read_seqbegin(&timekeeper.lock);

		*ts = timekeeper.xtime;

        // クロックソースからナノ秒を読み取る
        // 「秒」は timekeeper.xtime.tv_sec をそのまま返す様子
		nsecs = timekeeping_get_ns();

		/* If arch requires, add in gettimeoffset() */
		nsecs += arch_gettimeoffset();

   // ロックとれるまでループ
	} while (read_seqretry(&timekeeper.lock, seq));

    // struct timespec の計算用ルーチ
	timespec_add_ns(ts, nsecs);
}
EXPORT_SYMBOL(getnstimeofday);
```

> NTPデーモンは、時刻に狂いが出てくるとadjlimexシステムコールを利用してLinuxカーネルに時刻の修正を依頼します。

 [adjtimex(2)](http://linuxjm.sourceforge.jp/html/LDP_man-pages/man2/adjtimex.2.html)。し仕様が複雑なので atode ...

 ## 4.6 時刻管理の問題

 Linux内部時計 (xtime) の説明

  * 32bit での xtime (long型 = 4bytes) のオーバーフローの話
  * mtime, atime, ctime も xtime を使ってセットされている
  * CentOS5 のソースだと xtime はグローバル変数の **struct timespec**
```c
// include/linux/time.h
extern struct timespec xtime;
```
 * CentOS6.5 のソースだと **struct timekeeper** の .xtime で struct timespec
```c
/* Structure holding internal timekeeping values. */
struct timekeeper {

// ...

	/* The current time */
	struct timespec xtime;
```

> また、timesシステムコールの戻り値は、jiffiesを基に計算するため、時間とともに次第に大きくなり、長時間動作しているシステムではオーバフローすることがあります。

times(2) の定義は下記の通りにセットされている

```c
#include <sys/times.h>
clock_t times(struct tms *buf);

struct tms  {
    clock_t tms_utime;  /* user time */
    clock_t tms_stime;  /* system time */
    clock_t tms_cutime; /* user time of children */
    clock_t tms_cstime; /* system time of children */
};
```

> times() は過去のある時点から経過したクロック数 (clock tick) を返す。 この返り値は clock_t >型が取り得る範囲からオーバーフローするかもしれない。 エラーの場合、(clock_t) -1 が返され、 errno が適切に設定される。

過去のある時点とは???

times(2) の実装は下記の通りになっている。

  * `return (long) jiffies_64_to_clock_t(get_jiffies_64());` の部分がオーバーフローする可能性があるってことか?
  * 不思議インタフェースだ

```c
SYSCALL_DEFINE1(times, struct tms __user *, tbuf)
{
	if (tbuf) {
		struct tms tmp;

        // ->
        // プロセスのユーザ時間、システム時間、「終了を待っている」子プロセスのユーザ/システム時間をとってくる
		do_sys_times(&tmp);
		if (copy_to_user(tbuf, &tmp, sizeof(struct tms)))
			return -EFAULT;
	}
	force_successful_syscall_return();
	return (long) jiffies_64_to_clock_t(get_jiffies_64());
}

void do_sys_times(struct tms *tms)
{
	cputime_t tgutime, tgstime, cutime, cstime;

	spin_lock_irq(&current->sighand->siglock);
    // 全スレッドの時間を加算
	thread_group_times(current, &tgutime, &tgstime);
    // wait 待ち子プロセスのユー座時間とシステム時間
	cutime = current->signal->cutime;
	cstime = current->signal->cstime;
	spin_unlock_irq(&current->sighand->siglock);

	tms->tms_utime = cputime_to_clock_t(tgutime);
	tms->tms_stime = cputime_to_clock_t(tgstime);
	tms->tms_cutime = cputime_to_clock_t(cutime);
	tms->tms_cstime = cputime_to_clock_t(cstime);
}
```

サンプル書いてみたけど、jiffies 以外全部 0 になる。何か間違ってるかな

```c
#if 0
#!/bin/bash
CFLAGS="-O2 -std=gnu99 -W -Wall -fPIE -D_FORTIFY_SOURCE=2"
o=`basename $0`
o=".${o%.*}"
gcc ${CFLAGS} -o $o $0 && ./$o $*; exit
#endif

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/times.h>

int main()
{
	struct tms buf;
	clock_t jiffies = times(&buf);
	if (jiffies == -1) {
		perror("times");
		exit(1);
	}

	for (int i = 0; i < 100000000; i++) {
		pid_t pid = getpid();
		pid = pid * pid;
	}

	// > 全ての時間はクロック数で返される。
	printf("%ld, %ld, %ld, %ld, %ld\n",
	       jiffies,
	       buf.tms_utime,
	       buf.tms_stime,
	       buf.tms_cutime,
	       buf.tms_cstime);

	exit(0);
}
```

jiffies が使われている /proc は `/proc/<pid>/stat` か?

 * http://blog.livedoor.jp/kurt0027/archives/51870033.html

```
[vagrant@vagrant-centos65 ~]$ cat /proc/self/stat
3468 (cat) R 3449 3468 3449 34816 3468 4202496 215 0 0 0 0 0 0 0 20 0 1 0 7867478 6221824 141 18446744073709551615 4194304 4235780 140733613784960 140733218595128 140483486963504 0 0 0 0 0 0 0 17 3 0 0 0 0 0
```

`/proc/<pid>/stat` のハンドラは do_task_stat で実装されている

 * プロセスの生成時刻 `task_struct .real_start_time` を start_time (ticks = jiffies) に変えてる
 * jiffies がオーバーフローするとここが狂いそう 
   * real_start_time は fork の過程でセットされている
 ```c
	do_posix_clock_monotonic_gettime(&p->start_time);
	p->real_start_time = p->start_time;
	monotonic_to_bootbased(&p->real_start_time);
```

```c
static int do_task_stat(struct seq_file *m, struct pid_namespace *ns,
			struct pid *pid, struct task_struct *task, int whole)
{

// ...

    // real_start_time を jiffies に変える
	/* Temporary variable needed for gcc-2.96 */
	/* convert timespec -> nsec*/
	start_time =
		(unsigned long long)task->real_start_time.tv_sec * NSEC_PER_SEC
				+ task->real_start_time.tv_nsec;
	/* convert nsec -> ticks */
	start_time = nsec_to_clock_t(start_time);

	seq_printf(m, "%d (%s) %c %d %d %d %d %d %u %lu \
%lu %lu %lu %lu %lu %ld %ld %ld %ld %d 0 %llu %lu %ld %lu %lu %lu %lu %lu \
%lu %lu %lu %lu %lu %lu %lu %lu %d %d %u %u %llu %lu %ld\n",
		pid_nr_ns(pid, ns),
		tcomm,
		state,
		ppid,
		pgid,
		sid,
		tty_nr,
		tty_pgrp,
		task->flags,
		min_flt,
		cmin_flt,
		maj_flt,
		cmaj_flt,
		cputime_to_clock_t(utime),
		cputime_to_clock_t(stime),
		cputime_to_clock_t(cutime),
		cputime_to_clock_t(cstime),
		priority,
		nice,
		num_threads,
        // これ
		start_time,
		vsize,
		mm ? get_mm_rss(mm) : 0,
		rsslim,
		mm ? (permitted ? mm->start_code : 1) : 0,
		mm ? (permitted ? mm->end_code : 1) : 0,
		(permitted && mm) ? mm->start_stack : 0,
		esp,
		eip,
		/* The signal information here is obsolete.
		 * It must be decimal for Linux 2.0 compatibility.
		 * Use /proc/#/status for real-time signals.
		 */
		task->pending.signal.sig[0] & 0x7fffffffUL,
		task->blocked.sig[0] & 0x7fffffffUL,
		sigign      .sig[0] & 0x7fffffffUL,
		sigcatch    .sig[0] & 0x7fffffffUL,
		wchan,
		0UL,
		0UL,
		task->exit_signal,
		task_cpu(task),
		task->rt_priority,
		task->policy,
		(unsigned long long)delayacct_blkio_ticks(task),
		cputime_to_clock_t(gtime),
		cputime_to_clock_t(cgtime));
	if (mm)
		mmput(mm);
	return 0;
}
```