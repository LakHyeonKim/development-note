---
description: mac os 노트북에서 비교 해보자!!!
---

# 📒 MVC 환경과 WebFlux 환경 스케줄링 성능 비교

## Spring MVC vs Sprint WebFlux + Coroutines

> Spring MVC와 Spring WebFlux + Coroutines를 성능 비교입니다. WebFlux는 비동기, 논블로킹 방식으로 동작하며, Coroutines는 Kotlin의 비동기 프로그래밍 방식입니다. Spring WebFlux는 Kotlin reactive 프로그래밍을 지원하며, Coroutines를 사용하여 비동기 프로그래밍을 할 수 있습니다.

### 리엑티브 프로그래밍 <a href="#user-content" id="user-content"></a>

* 리액티브 프로그래밍은 비동기적 데이터 흐름과 변화 전파를 처리하기 위한 프로그래밍 패러다임입니다. 이벤트 중심의 시스템에서 데이터를 스트림으로 처리하며, 시스템의 반응성과 확장성을 향상시킬 수 있습니다.
* Java는 리액티브 프로그래밍의 표준 API는 Reactive Streams 스펙을 기반으로 합니다
* 2015년 9월에 Reactive Streams 1.0.0이 발표되었으며, Java9 [Flow](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/Flow.html)에 포함되었습니다.
* 주요 인터페이스: Publisher, Subscriber, Subscription, Processor
* [Armeria로 Reactive Streams와 놀자!](https://engineering.linecorp.com/ko/blog/reactive-streams-with-armeria-1) 라인 기술블로그에 잘 설명되어 있습니다.

1. **Publisher**\
   데이터를 제공하는 역할을 하며, 데이터 스트림을 생성합니다. Publisher는 Subscriber가 구독하면 데이터를 전달하며, 필요에 따라 구독 해지 및 상태를 관리합니다.
2. **Subscriber**\
   Publisher가 제공하는 데이터를 소비하는 역할을 합니다. 데이터 스트림을 처리하기 위해 onSubscribe, onNext, onError, onComplete 메서드를 구현해야 합니다.
3. **Subscription**\
   Publisher와 Subscriber 간의 구독 관계를 나타냅니다. Subscriber가 데이터 요청의 수량(backpressure)이나 구독 취소 등을 관리할 수 있도록 도와줍니다.
4. **Processor**\
   Publisher와 Subscriber의 중간에 위치하며, 데이터 스트림을 변환하거나 가공할 수 있습니다. Processor는 Publisher와 Subscriber의 기능을 모두 구현해야 합니다.

### Spring WebFlux <a href="#user-content-spring-webflux" id="user-content-spring-webflux"></a>

* [Spring WebFlux](https://docs.spring.io/spring-framework/reference/web/webflux.html) 는 [Flow](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/Flow.html) 인터페이스 기반으로 Reactive Stream 라이브러리로 구현되었습니다.
* Reactive Stream 는 2가지 publisher 를 지원합니다.
  * Mono: 0 또는 1개의 데이터를 처리합니다.
  * Flux: 0 또는 N개의 데이터를 처리합니다.
* 기본 내장 서버는 Tomcat 대신 Netty를 사용하며, 비동기, 논블로킹 방식으로 동작합니다.

### 스케줄링 테스트 설정 <a href="#user-content" id="user-content"></a>

* task 당 0\~200ms 로 수행시간이 랜덤하게 발생하도록 스케줄링 테스트를 진행합니다.
* fixedDelay 조건으로 실행하고 1000ms 간격입니다.
* 총 1000, 2000, 5000 테스크 개수를 증가하며 결과를 확인 합니다.
* MVC 경우 ThreadPoolTaskScheduler 를 사용하며, ThreadPool 크기는 200 입니다.
* WebFlux 경우 코루틴 기반으로 작성하였고, 코루틴 컨텍스트로 [Dispatchers.Default](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html) 를 사용하여 시스템 코어 수(테스트 환경 8코어) 만큼인 8개의 스레드를 사용합니다.

#### MVC 스케줄링 <a href="#user-content-mvc" id="user-content-mvc"></a>

* 구성 코드

```java

// ThreadPoolTaskScheduler 풀 개수 200

@Configuration
public class SchedulerConfig {

  @Bean
  public ThreadPoolTaskScheduler taskScheduler() {
    ThreadPoolTaskScheduler taskScheduler = new ThreadPoolTaskScheduler();
    taskScheduler.setPoolSize(200); // 스레드 풀 크기
    taskScheduler.setThreadNamePrefix("DynamicScheduler-");
    return taskScheduler;
  }

}

// 동적으로 스케줄링 작업 추가

@Slf4j
@Component
public class DynamicSchedulerService {

  private final ThreadPoolTaskScheduler taskScheduler;

  // 스케줄된 작업 저장소
  private final Map<String, ScheduledFuture<?>> scheduledTasks = new ConcurrentHashMap<>();

  // 작업별 실행 시간 기록
  private final Map<String, List<Long>> executionStats = new ConcurrentHashMap<>();

  @Autowired
  public DynamicSchedulerService(ThreadPoolTaskScheduler taskScheduler) {
    this.taskScheduler = taskScheduler;
  }

  public void addScheduledTask(String taskId, Long interval,
    Consumer<Map<String, List<Long>>> consumer) {
    Runnable task = () -> consumer.accept(executionStats);
    ScheduledFuture<?> scheduledFuture = taskScheduler.scheduleWithFixedDelay(task,
      Duration.ofMillis(interval));
    scheduledTasks.put(taskId, scheduledFuture);
  }

  public void cancelAllScheduledTasks() {
    scheduledTasks.keySet().forEach(this::cancelScheduledTask);
  }

  public void cancelScheduledTask(String taskId) {
    ScheduledFuture<?> scheduledTask = scheduledTasks.get(taskId);
    if (scheduledTask != null) {
      scheduledTask.cancel(true);
      scheduledTasks.remove(taskId);
//      log.info("Task {} has been cancelled.", taskId);
    } else {
      log.info("No task found with ID: {}", taskId);
    }
  }

  public void printTaskStatistics() {
    executionStats.keySet().forEach(this::printTaskStatistics);
  }

  public void printTaskStatistics(String taskId) {
    List<Long> timestamps = executionStats.get(taskId);
    if (timestamps == null || timestamps.size() < 2) {
      log.info("Not enough data to calculate statistics for task: {}", taskId);
      return;
    }

    List<Long> intervals = calculateIntervals(timestamps);
    log.info("Statistics for task: {}", taskId);
    log.info("Execution count: {}", timestamps.size());
    log.info("Execution intervals: {}", intervals);
    log.info("Average interval: {} ms", calculateAverage(intervals));
    log.info("Min interval: {} ms", intervals.stream().min(Long::compare).orElse(0L));
    log.info("Max interval: {} ms", intervals.stream().max(Long::compare).orElse(0L));
  }

  /**
   * 타임스탬프 간 간격 계산
   *
   * @param timestamps 실행 타임스탬프 리스트
   * @return 간격 리스트
   */
  private List<Long> calculateIntervals(List<Long> timestamps) {
    return timestamps.stream()
      .skip(1)
      .map(i -> i - timestamps.get(timestamps.indexOf(i) - 1))
      .toList();
  }

  /**
   * 평균 계산
   *
   * @param intervals 간격 리스트
   * @return 평균 값
   */
  private double calculateAverage(List<Long> intervals) {
    return intervals.stream()
      .mapToLong(Long::longValue)
      .average()
      .orElse(0);
  }
}

// 테스트 코드

@Slf4j
@SpringBootTest
class MvcApplicationTests {

  @Autowired
  private DynamicSchedulerService dynamicSchedulerService;

  @Test
  void contextLoads() throws InterruptedException {
    int size = 1000; // task 수량
    for (int i = 0; i < size; i++) {
      String taskId = "task-" + i;
      dynamicSchedulerService.addScheduledTask(taskId, 1000L, executionStats -> {
        long startTime = Clock.systemUTC().millis();
        long delay = Math.round(Math.random() * 200); // 작업 수행시간 0~200ms 랜덤
        log.info("Task {} running on thread {} delay: {}", taskId, Thread.currentThread().getName(),
          delay);
        executionStats.computeIfAbsent(taskId,
            k -> new java.util.concurrent.CopyOnWriteArrayList<>())
          .add(startTime);
        try {
          Thread.sleep(delay);
        } catch (InterruptedException e) {
          throw new RuntimeException(e);
        }
      });
    }

    Thread.sleep(120000L);

    dynamicSchedulerService.printTaskStatistics();
    dynamicSchedulerService.cancelAllScheduledTasks();
  }

}

```

#### WebFlux + Coroutines 스케줄링 <a href="#user-content-webflux-coroutines" id="user-content-webflux-coroutines"></a>

* 구성 코드

```kotlin

// 스케줄러 설정

@Component
class AdvancedCoroutineScheduler(
    private val jobStatusService: JobStatusService,
) {

    private val log = logger()

    companion object {
        private const val MIN_HOLD_TIME = 100L
        private const val HOLD_RATE = 0.9
    }

    private val scope = CoroutineScope(Dispatchers.Default + SupervisorJob())
    private val jobMap = ConcurrentHashMap<String, Job>()

    suspend fun scheduleTaskWithLock(
        databaseName: String,
        key: String,
        cronExpression: String? = null,
        interval: Long? = null,
        initialDelay: Long? = null,
        maxDeadlineMillis: Long,
        zoneId: ZoneId = ZoneId.systemDefault(),
        allowedDuplicationKey: Boolean = false,
        task: suspend () -> Unit
    ) {
        if (!allowedDuplicationKey && jobMap.containsKey(key)) {
            throw IllegalArgumentException("Task with key $key is already scheduled.")
        }
        val job = scope.launch {
            initialDelay?.let { delay(it) }
            when {
                cronExpression != null -> scheduleCronTaskWithLock(
                    coroutineContext,
                    databaseName = databaseName,
                    key = key,
                    cronExpression = cronExpression,
                    zoneId = zoneId,
                    maxDeadlineMillis = maxDeadlineMillis,
                    task = task
                )

                interval != null -> scheduleFixedIntervalTaskWithLock(
                    coroutineContext,
                    databaseName = databaseName,
                    key = key,
                    interval = interval,
                    maxDeadlineMillis = maxDeadlineMillis,
                    task = task
                )

                else -> throw IllegalArgumentException("You must specify one of cronExpression, interval")
            }
        }
        jobMap[key] = job
    }

    fun stopTask(key: String) {
        jobMap[key]?.cancel()
        jobMap.remove(key)
        log.info("Task with key $key stopped.")
    }

    fun stopAllTasks() {
        jobMap.values.forEach { it.cancel() }
        jobMap.clear()
        log.info("All tasks stopped.")
    }

    private suspend fun scheduleCronTaskWithLock(
        context: CoroutineContext,
        databaseName: String,
        key: String,
        cronExpression: String,
        zoneId: ZoneId,
        maxDeadlineMillis: Long,
        task: suspend () -> Unit
    ) {
        val cronTrigger = CronTrigger(cronExpression, zoneId)
        val triggerContext = SimpleTriggerContext()
        val scheduledExecutionTime = triggerContext.clock.instant()
        while (context.isActive) {
            try {
                val nextExecutionTime = cronTrigger.nextExecution(triggerContext)
                if (nextExecutionTime != null) {
                    val delayMillis = Duration.between(Instant.now(), nextExecutionTime).toMillis()
                        .coerceAtLeast(0)
                    delay(delayMillis)
                    val (_, actualExecutionTime, completionTime) = measureTime(triggerContext.clock) {
                        executeTaskWithLock(
                            databaseName = databaseName,
                            key = key,
                            maxDeadlineMillis = maxDeadlineMillis,
                            nextExecutionDelayMillis = delayMillis,
                            isCronTime = true,
                            task = task
                        )
                    }
                    triggerContext.update(
                        scheduledExecutionTime,
                        actualExecutionTime,
                        completionTime
                    )
                }
            } catch (e: Exception) {
                log.error("Error occurred while executing cron task with key $key", e)
            }
        }
    }

    private suspend fun scheduleFixedIntervalTaskWithLock(
        context: CoroutineContext,
        databaseName: String,
        key: String,
        interval: Long,
        maxDeadlineMillis: Long,
        task: suspend () -> Unit
    ) {
        while (context.isActive) {
            try {
                val delayMillis = interval.coerceAtLeast(0)
                executeTaskWithLock(
                    databaseName = databaseName,
                    key = key,
                    maxDeadlineMillis = maxDeadlineMillis,
                    nextExecutionDelayMillis = delayMillis,
                    isCronTime = false,
                    task = task
                )
                delay(delayMillis)
            } catch (e: Exception) {
                log.error("Error occurred while executing interval task with key $key", e)
            }
        }
    }

    private suspend fun executeTaskWithLock(
        databaseName: String,
        key: String,
        maxDeadlineMillis: Long,
        nextExecutionDelayMillis: Long,
        isCronTime: Boolean,
        task: suspend () -> Unit
    ) {
        val isAcquired = jobStatusService.acquireJobLock(
            databaseName, key, maxDeadlineMillis,
            if (isCronTime) MIN_HOLD_TIME else nextExecutionDelayMillis
        )
        if (!isAcquired) {
            log.debug("Failed to acquire lock for key: $key")
            return
        }
        val jobStatusType = try {
            withTimeout(maxDeadlineMillis) {
                task()
            }
            log.debug("Task completed for key: $key")
            JobStatusType.COMPLETED
        } catch (e: TimeoutCancellationException) {
            log.error("Task timed out for key: $key")
            JobStatusType.TIMEOUT
        } catch (e: Exception) {
            log.error("Error occurred while executing task for key: $key", e)
            JobStatusType.ERROR
        }
        jobStatusService.updateJonStatus(databaseName, key, jobStatusType)
    }

    private suspend fun <T> measureTime(
        clock: Clock,
        block: suspend () -> T
    ): Triple<T, Instant, Instant> {
        val start = clock.instant()
        val result = block()
        val end = clock.instant()
        return Triple(result, start, end)
    }
}

// 테스트 코드

class AdvancedCoroutineSchedulerTest(private val advancedCoroutineScheduler: AdvancedCoroutineScheduler) :
    BaseNodeTest() {
    init {
        val log = logger()

        fun analyseExecutionStats(
            delay: Long,
            cleaner: suspend () -> Unit,
            testBlock: suspend (executionStats: ConcurrentHashMap<String, MutableList<Long>>) -> Unit
        ) {
            runBlocking {
                // 통계 저장소
                val executionStats = ConcurrentHashMap<String, MutableList<Long>>()

                testBlock(executionStats)
                delay(delay)
                cleaner()

                // 통계 결과 확인
                executionStats.forEach { (key, timestamps) ->
                    log.info("Statistics for job: $key")
                    val intervals = timestamps.zipWithNext { a, b -> b - a }
                    log.info("Execution count: ${timestamps.size}")
                    log.info("Execution intervals: $intervals")
                    log.info("Average interval: ${intervals.average()} ms")
                    log.info("Min interval: ${intervals.minOrNull()} ms, Max interval: ${intervals.maxOrNull()} ms")
                }
            }
        }

        describe("AdvancedCoroutineScheduler test case 8") {
            it("interval 시간으로 1초 간격으로 1000개의 서로다른 job 을 2분간 수행동안 랜덤으로 실행한다. 동시에 3개의 작업이 실행되야 하고 각 작업은 100회 근처로 수행되어야 한다.") {
                val delayTime = 120000L
                val cleaner: suspend () -> Unit = {
                    log.info("Stopping the tasks")
                    advancedCoroutineScheduler.stopAllTasks()
                    log.info("Tasks stopped")
                }

                analyseExecutionStats(delayTime, cleaner) {
                    repeat(1000) { index ->
                        val key = "key-$index"
                        advancedCoroutineScheduler.scheduleTaskWithLock(
                            "databaseName",
                            key,
                            interval = 1000,
                            maxDeadlineMillis = 4000,
                            allowedDuplicationKey = true
                        ) {
                            val startTime = Clock.systemUTC().millis()
                            val delay = Math.round(Math.random() * 200) // 작업 수행시간 0~200ms 랜덤
                            log.info("Task $index running on thread ${Thread.currentThread().name}, delay: $delay")
                            it.computeIfAbsent(key) {
                                Collections.synchronizedList(
                                    mutableListOf()
                                )
                            }
                                .add(startTime)
                            delay(delay)
                        }
                    }
                }
            }
        }
    }
}

```

### 테스트 결과 <a href="#user-content" id="user-content"></a>

#### 1. 1000개의 작업을 1초 간격으로 2분간 수행한 결과입니다. <a href="#user-content-1-1000-1-2" id="user-content-1-1000-1-2"></a>

**MVC**

```
DynamicSchedulerService : Statistics for task: task-611
DynamicSchedulerService : Execution count: 109
DynamicSchedulerService : Execution intervals: [1120, 1128, 1194, 1113, 1040, 1131, 1138, 1029, 1016, 1186, 1169, 1071, 1134, 1016, 1078, 1201, 1090, 1072, 1130, 1050, 1106, 1126, 1009, 1181, 1033, 1029, 1039, 1013, 1082, 1161, 1054, 1171, 1025, 1064, 1083, 1123, 1171, 1046, 1073, 1122, 1066, 1011, 1173, 1075, 1154, 1157, 1048, 1166, 1174, 1096, 1138, 1061, 1052, 1176, 1127, 1188, 1184, 1017, 1018, 1111, 1128, 1148, 1199, 1190, 1137, 1141, 1002, 1148, 1168, 1050, 1147, 1159, 1136, 1026, 1058, 1043, 1089, 1134, 1013, 1097, 1016, 1004, 1188, 1036, 1162, 1162, 1149, 1035, 1143, 1113, 1109, 1098, 1003, 1131, 1092, 1115, 1166, 1137, 1165, 1070, 1195, 1190, 1081, 1183, 1111, 1015, 1089, 1191]
DynamicSchedulerService : Average interval: 1105.287037037037 ms
DynamicSchedulerService : Min interval: 1002 ms
DynamicSchedulerService : Max interval: 1201 ms
DynamicSchedulerService : Statistics for task: task-853
DynamicSchedulerService : Execution count: 110
DynamicSchedulerService : Execution intervals: [1179, 1200, 1167, 1054, 1017, 1048, 1032, 1098, 1149, 1045, 1183, 1068, 1044, 1132, 1054, 1108, 1027, 1004, 1067, 1108, 1105, 1168, 1144, 1128, 1029, 1113, 1011, 1077, 1120, 1013, 1171, 1057, 1025, 1002, 1024, 1103, 1112, 1121, 1146, 1078, 1130, 1043, 1049, 1020, 1179, 1181, 1036, 1060, 1146, 1037, 1126, 1102, 1009, 1145, 1100, 1124, 1196, 1001, 1180, 1162, 1116, 1124, 1062, 1070, 1146, 1062, 1083, 1172, 1173, 1184, 1083, 1088, 1157, 1132, 1082, 1100, 1124, 1066, 1090, 1009, 1041, 1186, 1141, 1089, 1038, 1046, 1133, 1160, 1008, 1012, 1068, 1167, 1081, 1031, 1171, 1057, 1137, 1033, 1130, 1041, 1126, 1195, 1060, 1090, 1116, 1044, 1111, 1060, 1075]
DynamicSchedulerService : Average interval: 1094.7431192660551 ms
DynamicSchedulerService : Min interval: 1001 ms
DynamicSchedulerService : Max interval: 1200 ms
DynamicSchedulerService : Statistics for task: task-610
DynamicSchedulerService : Execution count: 108
DynamicSchedulerService : Execution intervals: [1175, 1172, 1101, 1119, 1145, 1014, 1114, 1093, 1022, 1140, 1159, 1065, 1156, 1016, 1173, 1083, 1038, 1017, 1185, 1181, 1186, 1098, 1170, 1115, 1162, 1152, 1035, 1202, 1190, 1048, 1058, 1104, 1147, 1109, 1140, 1182, 1050, 1100, 1068, 1165, 1118, 1176, 1135, 1197, 1169, 1135, 1028, 1122, 1204, 1069, 1111, 1105, 1164, 1171, 1119, 1154, 1161, 1129, 1072, 1029, 1024, 1058, 1202, 1153, 1181, 1189, 1005, 1063, 1020, 1028, 1072, 1095, 1133, 1134, 1132, 1052, 1093, 1041, 1187, 1042, 1058, 1093, 1100, 1061, 1093, 1076, 1196, 1081, 1126, 1050, 1186, 1008, 1148, 1196, 1078, 1170, 1170, 1093, 1127, 1006, 1151, 1018, 1058, 1018, 1123, 1199, 1155]
DynamicSchedulerService : Average interval: 1111.766355140187 ms
DynamicSchedulerService : Min interval: 1005 ms
DynamicSchedulerService : Max interval: 1204 ms
DynamicSchedulerService : Statistics for task: task-852
DynamicSchedulerService : Execution count: 108
DynamicSchedulerService : Execution intervals: [1051, 1095, 1184, 1118, 1178, 1131, 1078, 1183, 1136, 1065, 1165, 1172, 1177, 1111, 1105, 1127, 1177, 1114, 1123, 1061, 1091, 1110, 1078, 1136, 1142, 1086, 1107, 1008, 1083, 1087, 1147, 1205, 1031, 1163, 1191, 1174, 1099, 1177, 1014, 1066, 1191, 1173, 1021, 1196, 1195, 1065, 1106, 1199, 1043, 1063, 1204, 1019, 1168, 1151, 1000, 1086, 1114, 1173, 1168, 1136, 1024, 1156, 1036, 1102, 1047, 1192, 1001, 1097, 1017, 1032, 1140, 1143, 1141, 1082, 1144, 1186, 1099, 1030, 1125, 1081, 1015, 1200, 1092, 1203, 1164, 1034, 1164, 1137, 1126, 1126, 1153, 1153, 1026, 1085, 1150, 1137, 1183, 1147, 1095, 1188, 1140, 1125, 1173, 1044, 1004, 1030, 1047]
DynamicSchedulerService : Average interval: 1114.3271028037384 ms
DynamicSchedulerService : Min interval: 1000 ms
DynamicSchedulerService : Max interval: 1205 ms
DynamicSchedulerService : Statistics for task: task-851
DynamicSchedulerService : Execution count: 110
DynamicSchedulerService : Execution intervals: [1084, 1009, 1019, 1068, 1057, 1204, 1188, 1189, 1103, 1175, 1147, 1111, 1105, 1049, 1006, 1008, 1028, 1120, 1086, 1187, 1175, 1046, 1138, 1149, 1070, 1155, 1029, 1008, 1187, 1086, 1107, 1073, 1036, 1115, 1114, 1176, 1083, 1055, 1031, 1099, 1046, 1047, 1015, 1075, 1069, 1049, 1123, 1033, 1065, 1178, 1096, 1152, 1086, 1183, 1201, 1049, 1174, 1018, 1139, 1006, 1009, 1088, 1105, 1171, 1161, 1134, 1033, 1137, 1045, 1098, 1071, 1147, 1180, 1113, 1194, 1093, 1078, 1085, 1113, 1175, 1027, 1064, 1073, 1155, 1167, 1050, 1145, 1096, 1120, 1062, 1192, 1030, 1153, 1114, 1155, 1024, 1003, 1001, 1127, 1079, 1029, 1056, 1050, 1163, 1142, 1106, 1184, 1113, 1078]
DynamicSchedulerService : Average interval: 1097.8623853211009 ms
DynamicSchedulerService : Min interval: 1001 ms
DynamicSchedulerService : Max interval: 1204 ms
```

* task 1000개 중 5개 결과 샘플
* 샘플 task-611, task-853, task-610, task-852, task-851 의 평균 인터벌 타임 1105ms
* 샘플 task-611, task-853, task-610, task-852, task-851 의 평균 실행 횟수는 109회
* 단순 실행 간격만 본다면 2분에 120 회가 호출되어야 하고, task 의 작업수행 시간을 고려한다면 평균 109는 이상적이라 할 수 있습니다.

**WebFlux + Coroutines**

```
[SchedulerTest.kt invokeSuspend(31)]: Statistics for job: key-990
[SchedulerTest.kt invokeSuspend(33)]: Execution count: 109
[SchedulerTest.kt invokeSuspend(34)]: Execution intervals: [1285, 1234, 1079, 1016, 1180, 1076, 1107, 1150, 1109, 1254, 1049, 1032, 1061, 1186, 1016, 1031, 1166, 1057, 1091, 1105, 1147, 1016, 1060, 1067, 1019, 1030, 1037, 1188, 1018, 1101, 1105, 1094, 1190, 1149, 1104, 1064, 1184, 1026, 1198, 1153, 1074, 1041, 1103, 1186, 1010, 1052, 1083, 1013, 1176, 1013, 1091, 1010, 1202, 1175, 1058, 1036, 1191, 1155, 1087, 1056, 1048, 1029, 1061, 1042, 1151, 1102, 1011, 1141, 1077, 1022, 1028, 1137, 1024, 1116, 1169, 1116, 1006, 1091, 1183, 1059, 1124, 1095, 1139, 1185, 1151, 1100, 1091, 1175, 1018, 1127, 1077, 1064, 1193, 1018, 1145, 1026, 1118, 1033, 1060, 1090, 1057, 1150, 1104, 1122, 1195, 1124, 1078, 1138]
[SchedulerTest.kt invokeSuspend(35)]: Average interval: 1098.6666666666667 ms
[SchedulerTest.kt invokeSuspend(36)]: Min interval: 1006 ms, Max interval: 1285 ms
[SchedulerTest.kt invokeSuspend(31)]: Statistics for job: key-991
[SchedulerTest.kt invokeSuspend(33)]: Execution count: 107
[SchedulerTest.kt invokeSuspend(34)]: Execution intervals: [1309, 1242, 1192, 1202, 1053, 1168, 1033, 1112, 1176, 1182, 1171, 1017, 1140, 1201, 1058, 1110, 1100, 1030, 1189, 1006, 1137, 1044, 1139, 1071, 1061, 1168, 1147, 1014, 1186, 1074, 1132, 1219, 1107, 1016, 1183, 1025, 1158, 1021, 1262, 1016, 1092, 1164, 1103, 1098, 1192, 1008, 1058, 1122, 1176, 1140, 1183, 1120, 1136, 1150, 1153, 1141, 1127, 1129, 1018, 1121, 1178, 1028, 1176, 1202, 1094, 1053, 1037, 1062, 1126, 1056, 1057, 1130, 1093, 1103, 1099, 1177, 1110, 1191, 1038, 1148, 1197, 1201, 1176, 1171, 1087, 1194, 1018, 1146, 1151, 1017, 1149, 1005, 1184, 1021, 1133, 1200, 1172, 1020, 1005, 1167, 1061, 1183, 1159, 1021, 1149, 1096]
[SchedulerTest.kt invokeSuspend(35)]: Average interval: 1118.3301886792453 ms
[SchedulerTest.kt invokeSuspend(36)]: Min interval: 1005 ms, Max interval: 1309 ms
[SchedulerTest.kt invokeSuspend(31)]: Statistics for job: key-510
[SchedulerTest.kt invokeSuspend(33)]: Execution count: 108
[SchedulerTest.kt invokeSuspend(34)]: Execution intervals: [1358, 1267, 1162, 1089, 1105, 1049, 1166, 1065, 1052, 1106, 1101, 1069, 1167, 1013, 1113, 1010, 1035, 1102, 1200, 1026, 1164, 1014, 1120, 1071, 1162, 1168, 1155, 1166, 1172, 1039, 1102, 1133, 1126, 1003, 1073, 1010, 1030, 1021, 1151, 1188, 1195, 1194, 1160, 1131, 1113, 1163, 1085, 1190, 1053, 1114, 1191, 1170, 1200, 1094, 1077, 1171, 1023, 1161, 1156, 1091, 1108, 1181, 1194, 1177, 1009, 1098, 1035, 1053, 1053, 1101, 1038, 1161, 1175, 1141, 1139, 1040, 1092, 1015, 1027, 1064, 1201, 1076, 1116, 1156, 1168, 1066, 1186, 1168, 1140, 1177, 1164, 1037, 1048, 1192, 1170, 1140, 1115, 1137, 1058, 1066, 1086, 1115, 1033, 1097, 1188, 1007, 1130]
[SchedulerTest.kt invokeSuspend(35)]: Average interval: 1113.9532710280373 ms
[SchedulerTest.kt invokeSuspend(36)]: Min interval: 1003 ms, Max interval: 1358 ms
[SchedulerTest.kt invokeSuspend(31)]: Statistics for job: key-752
[SchedulerTest.kt invokeSuspend(33)]: Execution count: 110
[SchedulerTest.kt invokeSuspend(34)]: Execution intervals: [1277, 1091, 1268, 1047, 1125, 1170, 1180, 1109, 1067, 1243, 1181, 1155, 1177, 1080, 1069, 1036, 1140, 1048, 1049, 1162, 1062, 1028, 1187, 1041, 1070, 1069, 1167, 1042, 1055, 1174, 1097, 1041, 1154, 1025, 1133, 1067, 1060, 1177, 1033, 1054, 1107, 1116, 1149, 1172, 1157, 1008, 1105, 1047, 1050, 1008, 1122, 1107, 1103, 1159, 1186, 1107, 1035, 1195, 1083, 1024, 1030, 1152, 1079, 1035, 1016, 1066, 1065, 1081, 1038, 1029, 1024, 1141, 1048, 1019, 1074, 1080, 1111, 1086, 1084, 1139, 1043, 1118, 1068, 1037, 1189, 1098, 1040, 1122, 1061, 1133, 1003, 1051, 1043, 1071, 1083, 1009, 1200, 1111, 1062, 1022, 1177, 1092, 1087, 1081, 1114, 1041, 1030, 1064, 1008]
[SchedulerTest.kt invokeSuspend(35)]: Average interval: 1092.7064220183486 ms
[SchedulerTest.kt invokeSuspend(36)]: Min interval: 1003 ms, Max interval: 1277 ms
[SchedulerTest.kt invokeSuspend(31)]: Statistics for job: key-994
[SchedulerTest.kt invokeSuspend(33)]: Execution count: 108
[SchedulerTest.kt invokeSuspend(34)]: Execution intervals: [1254, 1156, 1202, 1067, 1133, 1106, 1145, 1198, 1158, 1087, 1011, 1012, 1174, 1158, 1062, 1043, 1061, 1040, 1056, 1004, 1197, 1194, 1134, 1026, 1095, 1177, 1060, 1135, 1029, 1096, 1078, 1018, 1011, 1183, 1052, 1096, 1148, 1190, 1114, 1063, 1151, 1060, 1162, 1180, 1037, 1178, 1184, 1079, 1100, 1011, 1091, 1123, 1201, 1186, 1062, 1047, 1133, 1155, 1093, 1062, 1128, 1184, 1052, 1058, 1090, 1098, 1100, 1133, 1169, 1122, 1206, 1064, 1106, 1165, 1056, 1044, 1155, 1081, 1095, 1174, 1029, 1179, 1138, 1160, 1165, 1185, 1024, 1176, 1171, 1201, 1051, 1024, 1149, 1199, 1176, 1053, 1030, 1016, 1188, 1060, 1140, 1120, 1041, 1181, 1114, 1136, 1006]
[SchedulerTest.kt invokeSuspend(35)]: Average interval: 1111.3084112149534 ms
[SchedulerTest.kt invokeSuspend(36)]: Min interval: 1004 ms, Max interval: 1254 ms
```

* task 1000개 중 5개 결과 샘플
* 샘플 key-990, key-991, key-510, key-752, key-994 의 평균 인터벌 타임 1098ms
* 샘플 key-990, key-991, key-510, key-752, key-994 의 평균 실행 횟수는 108회
* 단순 실행 간격만 본다면 2분에 120 회가 호출되어야 하고, task 의 작업수행 시간을 고려한다면 평균 108는 이상적이라 할 수 있습니다.

#### 2. 2000개의 작업을 1초 간격으로 2분간 수행한 결과입니다. <a href="#user-content-2-2000-1-2" id="user-content-2-2000-1-2"></a>

**MVC**

```
DynamicSchedulerService: Statistics for task: task-1030
DynamicSchedulerService: Execution count: 109
DynamicSchedulerService: Execution intervals: [1062, 1198, 1091, 1179, 1108, 1028, 1115, 1080, 1164, 1119, 1002, 1141, 1179, 1126, 1155, 1027, 1020, 1024, 1029, 1204, 1055, 1015, 1182, 1006, 1050, 1033, 1118, 1180, 1084, 1025, 1058, 1097, 1064, 1149, 1058, 1163, 1001, 1067, 1198, 1135, 1010, 1094, 1087, 1125, 1145, 1012, 1122, 1126, 1090, 1010, 1109, 1193, 1162, 1100, 1090, 1129, 1181, 1005, 1026, 1078, 1041, 1191, 1169, 1181, 1164, 1104, 1086, 1065, 1089, 1005, 1108, 1018, 1147, 1152, 1154, 1116, 1125, 1097, 1146, 1112, 1187, 1164, 1078, 1192, 1060, 1082, 1192, 1085, 1108, 1124, 1150, 1026, 1205, 1090, 1183, 1095, 1136, 1068, 1180, 1017, 1125, 1034, 1131, 1128, 1139, 1139, 1178, 1039]
DynamicSchedulerService: Average interval: 1104.5185185185185 ms
DynamicSchedulerService: Min interval: 1001 ms
DynamicSchedulerService: Max interval: 1205 ms
DynamicSchedulerService: Statistics for task: task-1031
DynamicSchedulerService: Execution count: 109
DynamicSchedulerService: Execution intervals: [1162, 1059, 1194, 1111, 1207, 1001, 1133, 1167, 1059, 1185, 1038, 1081, 1196, 1108, 1164, 1195, 1123, 1194, 1105, 1050, 1056, 1168, 1182, 1129, 1031, 1048, 1092, 1033, 1166, 1061, 1038, 1071, 1069, 1175, 1160, 1019, 1159, 1032, 1013, 1093, 1156, 1157, 1160, 1116, 1030, 1046, 1067, 1033, 1179, 1044, 1137, 1035, 1057, 1141, 1158, 1038, 1162, 1058, 1107, 1133, 1073, 1116, 1103, 1014, 1170, 1078, 1032, 1055, 1029, 1199, 1071, 1194, 1101, 1037, 1108, 1194, 1010, 1018, 1140, 1172, 1063, 1195, 1112, 1120, 1033, 1100, 1141, 1168, 1108, 1124, 1132, 1109, 1155, 1121, 1033, 1122, 1195, 1204, 1163, 1029, 1172, 1125, 1139, 1199, 1088, 1088, 1014, 1123]
DynamicSchedulerService: Average interval: 1107.6851851851852 ms
DynamicSchedulerService: Min interval: 1001 ms
DynamicSchedulerService: Max interval: 1207 ms
DynamicSchedulerService: Statistics for task: task-1032
DynamicSchedulerService: Execution count: 108
DynamicSchedulerService: Execution intervals: [1010, 1099, 1125, 1088, 1155, 1141, 1202, 1071, 1101, 1017, 1178, 1057, 1191, 1075, 1151, 1175, 1019, 1069, 1083, 1161, 1168, 1201, 1178, 1084, 1171, 1014, 1099, 1059, 1018, 1014, 1152, 1129, 1181, 1122, 1126, 1006, 1178, 1117, 1073, 1042, 1160, 1016, 1071, 1122, 1124, 1054, 1005, 1056, 1047, 1116, 1119, 1160, 1030, 1149, 1202, 1150, 1068, 1076, 1146, 1056, 1085, 1184, 1044, 1075, 1094, 1090, 1166, 1167, 1134, 1194, 1114, 1158, 1035, 1090, 1027, 1107, 1165, 1174, 1110, 1134, 1040, 1184, 1166, 1050, 1061, 1177, 1138, 1084, 1060, 1069, 1053, 1096, 1173, 1192, 1147, 1177, 1049, 1123, 1065, 1174, 1102, 1111, 1076, 1187, 1148, 1085, 1089]
DynamicSchedulerService: Average interval: 1108.8785046728972 ms
DynamicSchedulerService: Min interval: 1005 ms
DynamicSchedulerService: Max interval: 1202 ms
DynamicSchedulerService: Statistics for task: task-1033
DynamicSchedulerService: Execution count: 110
DynamicSchedulerService: Execution intervals: [1067, 1061, 1116, 1209, 1032, 1121, 1195, 1184, 1077, 1157, 1078, 1010, 1185, 1064, 1073, 1084, 1025, 1150, 1036, 1048, 1087, 1157, 1045, 1085, 1031, 1089, 1092, 1147, 1019, 1100, 1044, 1005, 1193, 1120, 1087, 1151, 1154, 1020, 1026, 1029, 1027, 1044, 1151, 1002, 1187, 1071, 1040, 1014, 1058, 1079, 1024, 1159, 1094, 1027, 1094, 1169, 1097, 1136, 1132, 1042, 1199, 1106, 1042, 1093, 1054, 1026, 1103, 1025, 1067, 1066, 1159, 1023, 1088, 1048, 1089, 1037, 1029, 1013, 1082, 1149, 1041, 1077, 1034, 1047, 1193, 1132, 1127, 1044, 1061, 1024, 1005, 1046, 1047, 1178, 1155, 1073, 1144, 1031, 1045, 1141, 1202, 1051, 1120, 1197, 1070, 1147, 1185, 1059, 1196]
DynamicSchedulerService: Average interval: 1088.7155963302753 ms
DynamicSchedulerService: Min interval: 1002 ms
DynamicSchedulerService: Max interval: 1209 ms
DynamicSchedulerService: Statistics for task: task-859
DynamicSchedulerService: Execution count: 110
DynamicSchedulerService: Execution intervals: [1032, 1058, 1122, 1064, 1128, 1170, 1158, 1023, 1157, 1074, 1137, 1061, 1072, 1111, 1074, 1150, 1152, 1050, 1045, 1044, 1037, 1063, 1063, 1183, 1004, 1174, 1087, 1079, 1068, 1008, 1124, 1094, 1057, 1201, 1023, 1181, 1094, 1033, 1087, 1031, 1203, 1066, 1112, 1098, 1113, 1074, 1180, 1082, 1041, 1007, 1016, 1019, 1017, 1124, 1006, 1162, 1176, 1051, 1079, 1110, 1044, 1033, 1090, 1021, 1205, 1153, 1105, 1192, 1107, 1035, 1090, 1023, 1093, 1195, 1015, 1150, 1118, 1176, 1169, 1024, 1012, 1083, 1170, 1195, 1006, 1126, 1118, 1107, 1112, 1103, 1154, 1183, 1154, 1070, 1056, 1103, 1148, 1051, 1136, 1015, 1196, 1009, 1096, 1188, 1044, 1044, 1048, 1107, 1023]
DynamicSchedulerService: Average interval: 1093.6146788990825 ms
DynamicSchedulerService: Min interval: 1004 ms
DynamicSchedulerService: Max interval: 1205 ms
```

* task 2000개 중 5개 결과 샘플
* 샘플 task-1030, task-1031, task-1032, task-1033, task-859 의 평균 인터벌 타임 1100ms
* 샘플 task-1030, task-1031, task-1032, task-1033, task-859 의 평균 실행 횟수는 109회
* 단순 실행 간격만 본다면 2분에 120 회가 호출되어야 하고, task 의 작업수행 시간을 고려한다면 평균 109는 이상적이라 할 수 있습니다.

**WebFlux + Coroutines**

```
[SchedulerTest.kt invokeSuspend(31)]: Statistics for job: key-1720
[SchedulerTest.kt invokeSuspend(33)]: Execution count: 107
[SchedulerTest.kt invokeSuspend(34)]: Execution intervals: [1494, 1232, 1167, 1205, 1040, 1105, 1125, 1170, 1037, 1131, 1009, 1136, 1126, 1104, 1185, 1137, 1195, 1059, 1149, 1156, 1070, 1086, 1114, 1084, 1118, 1013, 1028, 1050, 1125, 1194, 1154, 1110, 1185, 1073, 1190, 1035, 1225, 1297, 1148, 1197, 1213, 1136, 1059, 1180, 1192, 1006, 1033, 1076, 1094, 1168, 1146, 1032, 1182, 1153, 1078, 1173, 1089, 1030, 1009, 1192, 1015, 1076, 1146, 1072, 1035, 1133, 1098, 1144, 1123, 1085, 1097, 1116, 1112, 1034, 1146, 1081, 1137, 1031, 1124, 1130, 1046, 1093, 1061, 1058, 1120, 1059, 1134, 1098, 1047, 1039, 1150, 1188, 1051, 1034, 1067, 1180, 1169, 1081, 1086, 1084, 1071, 1154, 1162, 1104, 1062, 1119]
[SchedulerTest.kt invokeSuspend(35)]: Average interval: 1114.632075471698 ms
[SchedulerTest.kt invokeSuspend(36)]: Min interval: 1006 ms, Max interval: 1494 ms
[SchedulerTest.kt invokeSuspend(31)]: Statistics for job: key-1721
[SchedulerTest.kt invokeSuspend(33)]: Execution count: 107
[SchedulerTest.kt invokeSuspend(34)]: Execution intervals: [1532, 1191, 1249, 1090, 1163, 1086, 1171, 1190, 1162, 1171, 1068, 1177, 1086, 1191, 1296, 1006, 1138, 1026, 1098, 1057, 1049, 1047, 1017, 1048, 1087, 1168, 1159, 1078, 1086, 1125, 1073, 1181, 1060, 1199, 1133, 1132, 1249, 1124, 1205, 1212, 1154, 1077, 1108, 1113, 1196, 1193, 1051, 1087, 1154, 1178, 1065, 1197, 1169, 1015, 1113, 1086, 1110, 1041, 1124, 1245, 1154, 1016, 1023, 1179, 1056, 1172, 1137, 1161, 1059, 1086, 1195, 1037, 1003, 1038, 1024, 1082, 1153, 1044, 1131, 1021, 1159, 1121, 1189, 1118, 1046, 1175, 1039, 1141, 1172, 1035, 1100, 1085, 1188, 1174, 1007, 1203, 1179, 1115, 1044, 1124, 1033, 1190, 1016, 1012, 1085, 1106]
[SchedulerTest.kt invokeSuspend(35)]: Average interval: 1119.6509433962265 ms
[SchedulerTest.kt invokeSuspend(36)]: Min interval: 1003 ms, Max interval: 1532 ms
[SchedulerTest.kt invokeSuspend(31)]: Statistics for job: key-830
[SchedulerTest.kt invokeSuspend(33)]: Execution count: 109
[SchedulerTest.kt invokeSuspend(34)]: Execution intervals: [1652, 1198, 1014, 1130, 1112, 1105, 1105, 1124, 1087, 1018, 1072, 1003, 1122, 1092, 1107, 1207, 1155, 1093, 1094, 1132, 1039, 1201, 1087, 1106, 1055, 1067, 1055, 1091, 1134, 1037, 1040, 1156, 1052, 1115, 1181, 1097, 1075, 1103, 1284, 1102, 1217, 1170, 1128, 1050, 1157, 1009, 1030, 1037, 1037, 1048, 1061, 1177, 1164, 1093, 1198, 1061, 1182, 1061, 1118, 1033, 1017, 1130, 1194, 1206, 1020, 1068, 1039, 1034, 1014, 1118, 1175, 1022, 1082, 1136, 1026, 1130, 1030, 1066, 1048, 1156, 1098, 1022, 1123, 1166, 1147, 1070, 1058, 1026, 1193, 1037, 1193, 1143, 1037, 1134, 1080, 1063, 1176, 1050, 1104, 1106, 1020, 1093, 1047, 1071, 1042, 1073, 1180, 1187]
[SchedulerTest.kt invokeSuspend(35)]: Average interval: 1103.5185185185185 ms
[SchedulerTest.kt invokeSuspend(36)]: Min interval: 1003 ms, Max interval: 1652 ms
[SchedulerTest.kt invokeSuspend(31)]: Statistics for job: key-833
[SchedulerTest.kt invokeSuspend(33)]: Execution count: 108
[SchedulerTest.kt invokeSuspend(34)]: Execution intervals: [1584, 1178, 1060, 1129, 1058, 1031, 1132, 1068, 1021, 1010, 1015, 1017, 1068, 1155, 1153, 1137, 1153, 1096, 1071, 1176, 1111, 1163, 1069, 1172, 1196, 1008, 1170, 1146, 1058, 1047, 1033, 1075, 1178, 1119, 1114, 1037, 1172, 1165, 1261, 1185, 1113, 1147, 1106, 1028, 1076, 1183, 1028, 1139, 1207, 1113, 1038, 1062, 1172, 1147, 1071, 1043, 1030, 1081, 1059, 1117, 1108, 1144, 1097, 1017, 1006, 1059, 1070, 1197, 1033, 1093, 1083, 1201, 1050, 1109, 1193, 1142, 1047, 1174, 1167, 1170, 1038, 1055, 1013, 1112, 1122, 1078, 1006, 1112, 1093, 1139, 1097, 1010, 1274, 1198, 1134, 1118, 1013, 1139, 1055, 1063, 1173, 1085, 1113, 1051, 1083, 1052, 1165]
[SchedulerTest.kt invokeSuspend(35)]: Average interval: 1107.4953271028037 ms
[SchedulerTest.kt invokeSuspend(36)]: Min interval: 1006 ms, Max interval: 1584 ms
[SchedulerTest.kt invokeSuspend(31)]: Statistics for job: key-834
[SchedulerTest.kt invokeSuspend(33)]: Execution count: 110
[SchedulerTest.kt invokeSuspend(34)]: Execution intervals: [1617, 1294, 1302, 1048, 1020, 1198, 1151, 1010, 1145, 1035, 1019, 1149, 1195, 1040, 1027, 1198, 1087, 1178, 1024, 1021, 1086, 1079, 1082, 1024, 1055, 1047, 1004, 1040, 1010, 1011, 1039, 1176, 1084, 1195, 1061, 1135, 1162, 1268, 1143, 1196, 1109, 1179, 1010, 1047, 1078, 1058, 1078, 1054, 1139, 1114, 1016, 1039, 1096, 1082, 1025, 1034, 1086, 1014, 1143, 1172, 1135, 1151, 1058, 1100, 1032, 1065, 1098, 1015, 1091, 1157, 1128, 1110, 1015, 1008, 1044, 1179, 1203, 1170, 1069, 1184, 1119, 1044, 1018, 1053, 1058, 1018, 1082, 1011, 1102, 1122, 1136, 1031, 1107, 1027, 1109, 1062, 1006, 1052, 1040, 1146, 1061, 1061, 1176, 1050, 1191, 1126, 1176, 1019, 1016]
[SchedulerTest.kt invokeSuspend(35)]: Average interval: 1095.954128440367 ms
[SchedulerTest.kt invokeSuspend(36)]: Min interval: 1004 ms, Max interval: 1617 ms
```

* task 2000개 중 5개 결과 샘플
* 샘플 key-1720, key-1721, key-830, key-833, key-834 의 평균 인터벌 타임 1114ms
* 샘플 key-1720, key-1721, key-830, key-833, key-834 의 평균 실행 횟수는 108회
* 단순 실행 간격만 본다면 2분에 120 회가 호출되어야 하고, task 의 작업수행 시간을 고려한다면 평균 108는 이상적이라 할 수 있습니다.

#### 3. 5000개의 작업을 1초 간격으로 2분간 수행한 결과입니다. <a href="#user-content-3-5000-1-2" id="user-content-3-5000-1-2"></a>

**MVC**

```
[DynamicSchedulerService: Statistics for task: task-1910
[DynamicSchedulerService: Execution count: 47
[DynamicSchedulerService: Execution intervals: [2498, 2558, 2671, 2638, 2613, 2529, 2615, 2546, 2658, 2498, 2479, 2644, 2516, 2565, 2496, 2597, 2553, 2620, 2634, 2598, 2602, 2614, 2522, 2469, 2488, 2650, 2568, 2560, 2551, 2572, 2588, 2586, 2670, 2534, 2470, 2471, 2572, 2638, 2524, 2516, 2524, 2534, 2652, 2639, 2707, 2535]
[DynamicSchedulerService: Average interval: 2571.3478260869565 ms
[DynamicSchedulerService: Min interval: 2469 ms
[DynamicSchedulerService: Max interval: 2707 ms
[DynamicSchedulerService: Statistics for task: task-4184
[DynamicSchedulerService: Execution count: 46
[DynamicSchedulerService: Execution intervals: [2632, 2597, 2576, 2555, 2549, 2623, 2514, 2542, 2574, 2656, 2520, 2608, 2520, 2679, 2577, 2617, 2520, 2606, 2513, 2606, 2575, 2608, 2605, 2609, 2529, 2569, 2486, 2496, 2600, 2553, 2586, 2520, 2561, 2636, 2528, 2513, 2511, 2639, 2583, 2637, 2599, 2682, 2605, 2538, 2503]
[DynamicSchedulerService: Average interval: 2574.5555555555557 ms
[DynamicSchedulerService: Min interval: 2486 ms
[DynamicSchedulerService: Max interval: 2682 ms
[DynamicSchedulerService: Statistics for task: task-4183
[DynamicSchedulerService: Execution count: 47
[DynamicSchedulerService: Execution intervals: [2512, 2481, 2605, 2508, 2582, 2534, 2623, 2438, 2502, 2609, 2535, 2633, 2568, 2588, 2670, 2563, 2602, 2559, 2545, 2514, 2652, 2609, 2612, 2487, 2543, 2522, 2606, 2502, 2465, 2646, 2659, 2651, 2569, 2583, 2493, 2447, 2495, 2682, 2508, 2581, 2640, 2534, 2522, 2646, 2660, 2566]
[DynamicSchedulerService: Average interval: 2566.3260869565215 ms
[DynamicSchedulerService: Min interval: 2438 ms
[DynamicSchedulerService: Max interval: 2682 ms
[DynamicSchedulerService: Statistics for task: task-4186
[DynamicSchedulerService: Execution count: 47
[DynamicSchedulerService: Execution intervals: [2627, 2618, 2585, 2661, 2609, 2617, 2481, 2551, 2536, 2570, 2606, 2567, 2622, 2540, 2675, 2504, 2639, 2465, 2491, 2639, 2484, 2501, 2466, 2542, 2475, 2510, 2462, 2583, 2597, 2654, 2565, 2631, 2546, 2547, 2607, 2487, 2549, 2508, 2604, 2578, 2621, 2577, 2585, 2532, 2642, 2587]
[DynamicSchedulerService: Average interval: 2566.1521739130435 ms
[DynamicSchedulerService: Min interval: 2462 ms
[DynamicSchedulerService: Max interval: 2675 ms
[DynamicSchedulerService: Statistics for task: task-899
[DynamicSchedulerService: Execution count: 47
[DynamicSchedulerService: Execution intervals: [2547, 2651, 2593, 2656, 2546, 2617, 2650, 2485, 2611, 2649, 2561, 2515, 2595, 2500, 2631, 2612, 2610, 2585, 2516, 2483, 2529, 2693, 2622, 2623, 2471, 2581, 2548, 2546, 2580, 2556, 2516, 2619, 2665, 2687, 2537, 2490, 2532, 2527, 2586, 2512, 2577, 2569, 2546, 2668, 2681, 2575]
[DynamicSchedulerService: Average interval: 2579.3260869565215 ms
[DynamicSchedulerService: Min interval: 2471 ms
[DynamicSchedulerService: Max interval: 2693 ms
```

* task 5000개 중 5개 결과 샘플
* 샘플 task-1910, task-4184, task-4183, task-4186, task-899 의 평균 인터벌 타임 2571ms
* 샘플 task-1910, task-4184, task-4183, task-4186, task-899 의 평균 실행 횟수는 47회
* 단순 실행 간격만 본다면 2분에 120 회가 호출되어야 하고, task 의 작업수행 시간을 고려한다면 평균 47는 처리량이 급격히 떨어지는 것을 볼 수 있습니다.

**WebFlux + Coroutines**

```
[SchedulerTest.kt invokeSuspend(31)] : Statistics for job: key-4996
[SchedulerTest.kt invokeSuspend(33)] : Execution count: 103
[SchedulerTest.kt invokeSuspend(34)] : Execution intervals: [2002, 1601, 1265, 1120, 1225, 1204, 1242, 1261, 1199, 1136, 1205, 1031, 1023, 1180, 1136, 1202, 1187, 1114, 1127, 1074, 1117, 1072, 1103, 1162, 1222, 1085, 1100, 1106, 1171, 1056, 1148, 1130, 1112, 1055, 1117, 1122, 1123, 1122, 1176, 1188, 1190, 1045, 1145, 1079, 1190, 1093, 1161, 1125, 1094, 1160, 1163, 1062, 1145, 1033, 1110, 1024, 1034, 1180, 1217, 1131, 1125, 1198, 1083, 1081, 1024, 1041, 1181, 1198, 1137, 1113, 1016, 1173, 1140, 1137, 1107, 1167, 1227, 1113, 1206, 1100, 1068, 1169, 1172, 1031, 1072, 1056, 1179, 1117, 1099, 1256, 1624, 1582, 1242, 1120, 1154, 1174, 1233, 1093, 1152, 1180, 1263, 1177]
[SchedulerTest.kt invokeSuspend(35)] : Average interval: 1158.6470588235295 ms
[SchedulerTest.kt invokeSuspend(36)] : Min interval: 1016 ms, Max interval: 2002 ms
[SchedulerTest.kt invokeSuspend(31)] : Statistics for job: key-2321
[SchedulerTest.kt invokeSuspend(33)] : Execution count: 104
[SchedulerTest.kt invokeSuspend(34)] : Execution intervals: [2180, 1485, 1109, 1157, 1092, 1091, 1032, 1262, 1223, 1197, 1051, 1025, 1077, 1058, 1061, 1055, 1112, 1170, 1224, 1274, 1135, 1100, 1155, 1113, 1019, 1129, 1137, 1189, 1041, 1172, 1033, 1153, 1046, 1099, 1047, 1191, 1044, 1103, 1170, 1271, 1229, 1194, 1196, 1126, 1131, 1111, 1106, 1091, 1084, 1245, 1145, 1032, 1155, 1070, 1147, 1162, 1112, 1139, 1200, 1164, 1118, 1089, 1029, 1189, 1037, 1116, 1172, 1112, 1160, 1213, 1166, 1119, 1146, 1065, 1171, 1020, 1103, 1218, 1093, 1139, 1037, 1133, 1087, 1161, 1086, 1137, 1101, 1199, 1179, 1127, 1115, 1515, 1455, 1228, 1044, 1096, 1196, 1208, 1092, 1131, 1181, 1188, 1248]
[SchedulerTest.kt invokeSuspend(35)] : Average interval: 1150.873786407767 ms
[SchedulerTest.kt invokeSuspend(36)] : Min interval: 1019 ms, Max interval: 2180 ms
[SchedulerTest.kt invokeSuspend(31)] : Statistics for job: key-3652
[SchedulerTest.kt invokeSuspend(33)] : Execution count: 103
[SchedulerTest.kt invokeSuspend(34)] : Execution intervals: [1975, 1592, 1171, 1083, 1069, 1146, 1242, 1083, 1191, 1238, 1101, 1055, 1204, 1221, 1050, 1059, 1178, 1171, 1116, 1241, 1108, 1135, 1025, 1070, 1026, 1126, 1116, 1092, 1095, 1223, 1203, 1187, 1181, 1031, 1017, 1013, 1128, 1339, 1067, 1116, 1131, 1112, 1161, 1068, 1180, 1114, 1128, 1189, 1125, 1213, 1039, 1196, 1072, 1206, 1189, 1182, 1033, 1131, 1086, 1209, 1082, 1048, 1236, 1118, 1116, 1043, 1165, 1157, 1165, 1179, 1138, 1125, 1036, 1080, 1270, 1082, 1069, 1128, 1187, 1191, 1121, 1183, 1188, 1029, 1126, 1037, 1210, 1113, 1118, 1228, 1688, 1404, 1012, 1275, 1176, 1226, 1058, 1135, 1185, 1176, 1256, 1257]
[SchedulerTest.kt invokeSuspend(35)] : Average interval: 1157.4901960784314 ms
[SchedulerTest.kt invokeSuspend(36)] : Min interval: 1012 ms, Max interval: 1975 ms
[SchedulerTest.kt invokeSuspend(31)] : Statistics for job: key-4997
[SchedulerTest.kt invokeSuspend(33)] : Execution count: 103
[SchedulerTest.kt invokeSuspend(34)] : Execution intervals: [2085, 1456, 1327, 1168, 1035, 1276, 1155, 1157, 1035, 1055, 1068, 1211, 1120, 1121, 1017, 1026, 1067, 1234, 1159, 1140, 1225, 1130, 1148, 1046, 1099, 1093, 1057, 1080, 1200, 1135, 1045, 1047, 1054, 1200, 1136, 1200, 1317, 1207, 1237, 1205, 1046, 1077, 1039, 1072, 1237, 1181, 1035, 1210, 1166, 1160, 1255, 1130, 1109, 1193, 1031, 1120, 1142, 1055, 1169, 1201, 1151, 1186, 1178, 1164, 1207, 1068, 1027, 1180, 1176, 1169, 1241, 1208, 1011, 1207, 1040, 1160, 1082, 1163, 1192, 1193, 1123, 1102, 1065, 1098, 1172, 1177, 1088, 1190, 1056, 1176, 1474, 1545, 1265, 1133, 1223, 1213, 1117, 1041, 1050, 1166, 1182, 1074]
[SchedulerTest.kt invokeSuspend(35)] : Average interval: 1158.1764705882354 ms
[SchedulerTest.kt invokeSuspend(36)] : Min interval: 1011 ms, Max interval: 2085 ms
[SchedulerTest.kt invokeSuspend(31)] : Statistics for job: key-12
[SchedulerTest.kt invokeSuspend(33)] : Execution count: 103
[SchedulerTest.kt invokeSuspend(34)] : Execution intervals: [2205, 1406, 1378, 1261, 1073, 1097, 1214, 1268, 1153, 1063, 1133, 1035, 1026, 1049, 1124, 1164, 1070, 1085, 1245, 1263, 1063, 1150, 1124, 1133, 1149, 1163, 1185, 1054, 1011, 1141, 1042, 1159, 1072, 1054, 1022, 1169, 1094, 1193, 1236, 1260, 1092, 1075, 1251, 1224, 1059, 1098, 1203, 1229, 1230, 1165, 1178, 1185, 1078, 1195, 1036, 1204, 1067, 1173, 1034, 1060, 1239, 1116, 1121, 1185, 1123, 1167, 1114, 1141, 1229, 1084, 1170, 1103, 1115, 1131, 1109, 1174, 1159, 1152, 1114, 1052, 1196, 1158, 1119, 1168, 1207, 1195, 1129, 1124, 1170, 1238, 1784, 1524, 1397, 1213, 1123, 1163, 1142, 1048, 1142, 1067, 1162, 1243]
[SchedulerTest.kt invokeSuspend(35)] : Average interval: 1167.9607843137255 ms
[SchedulerTest.kt invokeSuspend(36)] : Min interval: 1011 ms, Max interval: 2205 ms
```

* task 5000개 중 5개 결과 샘플
* 샘플 key-4996, key-2321, key-3652, key-4997, key-12 의 평균 인터벌 타임 1158ms
* 샘플 key-4996, key-2321, key-3652, key-4997, key-12 의 평균 실행 횟수는 103회
* 단순 실행 간격만 본다면 2분에 120 회가 호출되어야 하고, task 의 작업수행 시간을 고려한다면 평균 103는 다소 부족하지만 이상적이라 할 수 있습니다.

### 해석 <a href="#user-content" id="user-content"></a>

* 1000, 2000개의 작업을 1초 간격으로 2분간 수행한 결과를 보면, MVC 와 WebFlux + Coroutines 의 성능 차이는 거의 없습니다.
* 5000개의 작업을 1초 간격으로 2분간 수행한 결과를 보면, MVC 와 WebFlux + Coroutines 의 성능 차이가 두드러집니다.
* 심지어 WebFlux + Coroutines 는 각 task 를 수행하기전 mongodb lock 을 획득하고, 수행 후 lock 을 해제하는 작업을 수행하였습니다.

### 참고 <a href="#user-content" id="user-content"></a>

* **기존 동기방식 스케줄링 라이브러리인 quartz, spring scheduler 를 사용하지 않은 이유는 해당 스케줄링 쓰레드가 차단되어 job 이 많아지면 작업 전체적인 처리량이 떨어 질 수 있기 때문입니다.**
* MVC, WebFlux + Coroutines 성능은 요청의 단건 처리는 차이가 거의 없습니다. 하지만, 처리량이 많아질수록 Webflux 는 빛을 발합니다.
* 쓰레드가 차단되기 쉬운 로직인 I/O 작업, 외부 API 호출, DB 조회 등에서 압도적인 성능차이를 보입니다.
* WebFlux + Coroutines 는 이런 스케줄링 작업 뿐만 아니라 대량의 요청이 들어오더라도 역압(backpressure)을 통해 OOM 이나 서버 다운을 방지할 수 있습니다.
* 이는 jvm 대기 메모리를 줄이는 효과가 있고, 요청이 폭발적으로 증가하더라도 균일하게 안정적인 처리가 가능합니다.
