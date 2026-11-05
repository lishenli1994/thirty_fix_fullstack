全栈 + MySQL 面试复习完整笔记
说明

本文档整理了《全栈从 0 到 1：只讲代码版》和 MySQL 系列课程（建表 → 高可用），每课的讲解、练习和参考答案都完整保留。
下面的代码都是示意代码，用来讲清楚原理，不是你项目里的源码。面试时不要说"我们项目就是这样写的"。
标着 【待确认】 的内容，要先和你的真实项目核对。标着 【演练方案】 的内容，不能说成项目里已经实施。
课程里有不少面试话术用了"我们……"的口吻（比如"我们报工表按月分区""我们每天用 XtraBackup 备份"）。这些都是演练话术，用之前要按项目实际情况改写。没做过的，改成"如果要做，我会这样设计"。
不要编数字。 没有压测数据，就不说"百万级""从 3 秒优化到 50 毫秒"。只说自己实验里实际跑出来的结果。
两套示意表结构：全栈部分用 work_order + report_record，数据库课程用 work_report。真实的表名和字段以项目为准【待确认】。
目录
第一部分：全栈从 0 到 1（只讲代码版）
第 0 章 一次请求经过哪些代码
后端 B1～B13
前端 F1～F8
Copilot C1～C4
交付
自测
第二部分：MySQL 系列课程
第 1 课 建表（附补充题：同一工序只能有一条待确认）
第 2 课 Mapper 条件更新
第 3 课 事务与 @Transactional 失效场景
第 4 课 索引与 EXPLAIN（附真实环境解读）
第 4 课（下）慢查询日志与线上慢 SQL 定位
第 5 课 SQL 改写：深分页、OR 和 IN、隐式类型转换
第 6 课 JOIN 是怎么执行的
第 7 课 COUNT、GROUP BY、ORDER BY 优化
第 8 课 事务与 MVCC
第 9 课 InnoDB 锁
第 10 课 redo log、undo log、binlog 与两阶段提交
第 11 课 大表治理：分区、归档与分库分表
第 12 课 Online DDL 与 MDL 锁
第 13 课 备份与恢复
第 14 课 监控与故障诊断
第 15 课 高可用与架构
附录
A. 待完成的动手任务
B. 交给 GPT 继续讲的提示词
C. 参考链接
第一部分：全栈从 0 到 1（只讲代码版）
主线：先用一个功能（LeMES 报工确认）把"数据库 → 后端 → 前端"完整讲一遍，再逐个讲其他功能。

第 0 章：一次请求经过哪些代码
text
浏览器点「确认报工」
→ 前端组件 confirm()                    // Vue
→ axios 封装：带上 token，统一处理错误   // request.ts
→ Nginx 转发 /api 请求
→ Controller：接收参数、检查权限          // Spring MVC
→ Service：业务规则 + 事务                // @Transactional
→ Mapper：执行 SQL                        // MyBatis
→ MySQL：用条件更新防止超报
← 返回统一格式 { code, msg, data }
← 拦截器：成功取出 data，失败弹出提示
← 组件刷新列表

后端（Spring Boot + MyBatis）
B1. 先建表
sql
CREATE TABLE work_order (
id           BIGINT PRIMARY KEY AUTO_INCREMENT,
order_no     VARCHAR(32) NOT NULL UNIQUE,
status       TINYINT     NOT NULL,          -- 0未开工 1生产中 2完工
plan_qty     INT         NOT NULL,          -- 计划数量
reported_qty INT         NOT NULL DEFAULT 0,-- 已确认的报工数量
created_at   DATETIME    NOT NULL,
KEY idx_status_created (status, created_at) -- 工单列表：按状态筛选、按时间排序
);

CREATE TABLE report_record (
id            BIGINT PRIMARY KEY AUTO_INCREMENT,
work_order_id BIGINT  NOT NULL,
qty           INT     NOT NULL,
status        TINYINT NOT NULL,   -- 0待确认 1已确认 2已拒绝
reporter_id   BIGINT  NOT NULL,   -- 提交报工的工人
confirmer_id  BIGINT,             -- 确认的组长
created_at    DATETIME NOT NULL
);

B2. DTO 和参数校验：管"输入合不合法"
java
@Data
public class SubmitReportReq {
@NotNull(message = "工单不能为空")
private Long workOrderId;

    @NotNull @Min(value = 1, message = "数量必须大于0")
    private Integer qty;          // 这里只管 qty > 0；会不会超过计划量，由数据库条件更新来管
}

B3. 统一返回格式和全局异常处理
java
@Data @AllArgsConstructor
public class R<T> {
private int code;      // 0 表示成功
private String msg;
private T data;
public static <T> R<T> ok(T data) { return new R<>(0, "ok", data); }
public static R<Void> ok()        { return new R<>(0, "ok", null); }
public static R<Void> fail(int code, String msg) { return new R<>(code, msg, null); }
}

// 业务异常继承 RuntimeException → Spring 默认会回滚事务
public class BizException extends RuntimeException {
private final int code;
public BizException(int code, String msg) { super(msg); this.code = code; }
public int getCode() { return code; }
}

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {
@ExceptionHandler(BizException.class)
public R<Void> biz(BizException e) { return R.fail(e.getCode(), e.getMessage()); }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public R<Void> valid(MethodArgumentNotValidException e) {
        return R.fail(400, e.getBindingResult().getFieldError().getDefaultMessage());
    }

    @ExceptionHandler(Exception.class)
    public R<Void> other(Exception e) {
        log.error("未处理异常", e);          // 详细错误只写进服务端日志
        return R.fail(500, "系统繁忙");      // 前端只看到约定好的提示
    }
}

B4. Mapper：两条条件更新
java
@Mapper
public interface ReportMapper {

    @Select("SELECT * FROM report_record WHERE id = #{id}")
    ReportRecord findById(Long id);

    // 只有"待确认"的报工才能改成已确认，防止同一条报工被确认两次
    @Update("UPDATE report_record SET status = 1, confirmer_id = #{uid} " +
            "WHERE id = #{id} AND status = 0")
    int markConfirmed(@Param("id") Long id, @Param("uid") Long uid);

    // 防超报：判断和累加放在同一条 SQL 里完成
    @Update("UPDATE work_order SET reported_qty = reported_qty + #{qty} " +
            "WHERE id = #{orderId} AND reported_qty + #{qty} <= plan_qty")
    int addReported(@Param("orderId") Long orderId, @Param("qty") int qty);
}

推演：计划 100，已报 90，A 和 B 同时各确认 8。

A 先拿到这一行的行锁，90 + 8 = 98 ≤ 100，更新成功，返回 1。
B 等 A 释放行锁后，按最新值 98 判断：98 + 8 > 100，返回 0。
返回 0 表示"一行都没改"，代码必须把它当作失败处理。
B5. Service：业务规则和事务
java
@Service
@RequiredArgsConstructor
public class ReportService {
private final ReportMapper reportMapper;

    // 【演练方案】两步放在同一个事务里；原 LeMES 是否这样做【待确认】
    @Transactional(rollbackFor = Exception.class)   // 受检异常也回滚
    public void confirm(Long reportId, Long leaderId) {
        ReportRecord r = reportMapper.findById(reportId);
        if (r == null) throw new BizException(404, "报工不存在");

        if (reportMapper.markConfirmed(reportId, leaderId) == 0)
            throw new BizException(409, "该报工已被处理");

        if (reportMapper.addReported(r.getWorkOrderId(), r.getQty()) == 0)
            throw new BizException(409, "超出计划数量");   // 抛出异常 → 上一步的状态修改一起回滚
    }
}

对着上面的代码，看三个常见坑：

java
// 坑1：影响 0 行只 return false，不抛异常 → 事务不会回滚，状态已经被改成"已确认"
if (reportMapper.addReported(...) == 0) return false;   // ❌

// 坑2：同一个类里用 this 调用，不经过 Spring 代理 → confirm 上的 @Transactional 不生效
public void batchConfirm(List<Long> ids) { ids.forEach(id -> this.confirm(id, 1L)); }  // ❌
// 改法：把 confirm 放到另一个 Bean 里注入调用，或者给外层方法加事务

// 坑3：没写 rollbackFor 时，抛出 IOException（受检异常）默认不回滚
@Transactional
public void x() throws IOException { update(); throw new IOException(); }  // ❌ 不保证回滚

B6. Controller 和权限
java
// 注意：@PreAuthorize 要在配置类上开启 @EnableMethodSecurity 才生效
@RestController
@RequestMapping("/api/reports")
@RequiredArgsConstructor
public class ReportController {
private final ReportService reportService;

    @PostMapping
    public R<Long> submit(@RequestBody @Valid SubmitReportReq req,
                          @AuthenticationPrincipal LoginUser user) {
        return R.ok(reportService.submit(req, user.getId()));
    }

    // RBAC：给"组长"角色分配 report:confirm 权限；前端隐藏按钮只是为了体验，这里才是真正的拦截
    @PreAuthorize("hasAuthority('report:confirm')")
    @PostMapping("/{id}/confirm")
    public R<Void> confirm(@PathVariable Long id, @AuthenticationPrincipal LoginUser user) {
        reportService.confirm(id, user.getId());
        return R.ok();
    }
}

RBAC 的数据结构（示意）：

sql
-- 用户 → 角色 → 权限，多个用户共用一套角色权限
user_role(user_id, role_id)
role_perm(role_id, perm_code)   -- 例如 ('组长', 'report:confirm')

B7. AOP 日志切面
java
@Slf4j
@Aspect
@Component
public class ApiLogAspect {
@Around("@within(org.springframework.web.bind.annotation.RestController)")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
long start = System.currentTimeMillis();
try {
return pjp.proceed();
} finally {
log.info("{} 耗时 {}ms", pjp.getSignature().toShortString(),
System.currentTimeMillis() - start);
}
}
}

B8. 工单列表：联合索引 + 延迟关联
xml
<select id="pageByStatus" resultType="WorkOrder">
SELECT w.*
FROM work_order w
JOIN (
-- 子查询只在索引 (status, created_at) 上取 id（InnoDB 二级索引里自带主键），不需要回表
SELECT id FROM work_order
WHERE status = #{status}
ORDER BY created_at DESC, id DESC
LIMIT #{offset}, #{size}
) t ON w.id = t.id
ORDER BY w.created_at DESC, w.id DESC
</select>

好处：只有当前页的 size 行需要回表读取完整字段。
局限：offset 前面的行仍然要在索引上扫过去，深分页的成本只是降低了，并没有消失。
没有正式的压测数字，不要编"百万级""毫秒级"。是否做过 EXPLAIN【待确认】。
B9. Redis 缓存字典（Cache Aside）
java
@Service
@RequiredArgsConstructor
public class DictService {
private final DictMapper dictMapper;
private final RedisTemplate<String, Object> redis;

    @SuppressWarnings("unchecked")
    public List<DictItem> list(String type) {
        String key = "dict:" + type;
        Object cached = redis.opsForValue().get(key);
        if (cached != null) return (List<DictItem>) cached;          // 1. 缓存命中直接返回

        List<DictItem> list = dictMapper.listByType(type);            // 2. 未命中则查库
        redis.opsForValue().set(key, list, Duration.ofMinutes(30));   // 3. 写回缓存，并设置过期时间作兜底
        return list;
    }

    @Transactional
    public void update(DictItem item) {
        dictMapper.update(item);
        // 等事务提交后再删除缓存，避免删早了又被旧数据回填
        TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
            @Override public void afterCommit() { redis.delete("dict:" + item.getType()); }
        });
    }
}
// 原项目实际用的失效策略【待确认】

B10. Quartz 定时任务
java
@DisallowConcurrentExecution          // 上一次没跑完，不会启动下一次
public class DailyStatJob extends QuartzJobBean {
@Autowired private StatService statService;

    @Override
    protected void executeInternal(JobExecutionContext ctx) {
        statService.buildDaily(LocalDate.now().minusDays(1));  // 具体业务逻辑在 Service 里
    }
}

@Configuration
public class QuartzConfig {
@Bean public JobDetail dailyStatJob() {
return JobBuilder.newJob(DailyStatJob.class).withIdentity("dailyStat").storeDurably().build();
}
@Bean public Trigger dailyStatTrigger(JobDetail dailyStatJob) {
return TriggerBuilder.newTrigger().forJob(dailyStatJob)
.withSchedule(CronScheduleBuilder.cronSchedule("0 0 1 * * ?"))  // 每天 1 点；Quartz 只负责"何时触发"
.build();
}
}
// 项目当时是单实例还是集群【待确认】

B11. 极维度：ERP / WMS 的 DTO 映射
java
// 外部系统字段 → MES 内部字段
@Component
@RequiredArgsConstructor
public class ErpMaterialConverter {
private final CodeMappingMapper codeMapping;   // 外部编码与内部编码的映射表

    public Material toMes(ErpMaterialDTO dto) {
        String innerCode = codeMapping.findInner("ERP", dto.getMatnr());
        if (innerCode == null)
            throw new BizException(422, "物料编码未映射: " + dto.getMatnr());  // 先查映射，不要直接猜是哪边丢了数据
        Material m = new Material();
        m.setCode(innerCode);
        m.setName(dto.getMaktx());
        m.setUnit(dto.getMeins());
        return m;
    }
}

批次追溯的核心是一张关联表（示意，真实表结构【待确认】）：

sql
-- 工序记录了"哪道工序投入了哪个原料批次、产出了哪个批次"
batch_relation(output_batch, input_batch, work_order_id, process_id, qty)
-- 从成品批次出发，递归查询 input_batch，就能追溯到原料批次

B12. Kafka 消费者：实际做法与改进方案
java
// 【你的实际做法】先在 Redis 标记，再写库（TTL 是否设置、设置多长【待确认】）
@KafkaListener(topics = "device-data", groupId = "mes")
public void onMessage(DeviceMsg msg) {
Boolean first = redis.opsForValue()
.setIfAbsent("idem:" + msg.getMsgId(), "1", Duration.ofDays(1));  // 相当于 SET NX + 过期时间
if (!Boolean.TRUE.equals(first)) return;     // 已有 key，当作已处理
deviceDataMapper.insert(toEntity(msg));      // ⚠ 如果在这一步之前进程宕机：key 已经存在，数据却没写进去 → 重投时被跳过，消息漏处理
}

java
// 【演练方案】数据库唯一约束 + 同一事务 + 手动提交 offset
// 配置：spring.kafka.listener.ack-mode=manual
// 建表：consumed_msg(msg_id VARCHAR(64) PRIMARY KEY)

@KafkaListener(topics = "device-data", groupId = "mes")
public void onMessage(DeviceMsg msg, Acknowledgment ack) {
try {
deviceService.save(msg);                 // 必须是另一个 Bean，事务才会生效
} catch (DuplicateKeyException e) {
// 唯一约束冲突 = 之前已经处理成功，直接跳过
}
ack.acknowledge();                           // 数据库提交之后才提交 offset
}

@Service
public class DeviceService {
@Transactional
public void save(DeviceMsg msg) {
consumedMsgMapper.insert(msg.getMsgId());   // 去重记录
deviceDataMapper.insert(toEntity(msg));     // 业务数据；两者要么都成功，要么一起回滚
}
}

B13. Redis 分布式锁（课程实验）
java
String token = UUID.randomUUID().toString();
Boolean ok = redis.opsForValue().setIfAbsent("lock:order:1", token, Duration.ofSeconds(30)); // SET NX PX：加锁和设置过期时间一步完成
if (Boolean.TRUE.equals(ok)) {
try { doBiz(); }
finally {
// 比较值和删除放在 Lua 脚本里原子执行，避免误删别人的锁
String lua = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
redis.execute(new DefaultRedisScript<>(lua, Long.class), List.of("lock:order:1"), token);
}
}
// 锁仍可能在业务执行期间过期 → 最终的数据正确性还要靠数据库条件更新或 fencing token

前端（Vue 3 + TypeScript）
F1. axios 封装：所有请求的入口
ts
// src/utils/request.ts
import axios from 'axios'
import { ElMessage } from 'element-plus'
import { getToken, toLogin } from './auth'

const http = axios.create({ baseURL: '/api', timeout: 15000 })

http.interceptors.request.use(cfg => {
const t = getToken()                                // JWT 存在哪里【待确认】
if (t) cfg.headers.Authorization = `Bearer ${t}`
return cfg
})

http.interceptors.response.use(
res => {
const { code, msg, data } = res.data              // 对应后端的 R<T>
if (code !== 0) {
ElMessage.error(msg)
return Promise.reject(new Error(msg))
}
return data                                       // 业务代码直接拿到 data
},
err => {
if (err.response?.status === 401) toLogin()       // 未登录：跳转到认证中心
else ElMessage.error('网络异常')
return Promise.reject(err)
}
)
export default http

F2. OAuth2 授权码登录（你负责的前端部分）
ts
// src/utils/auth.ts
export function toLogin() {
const state = crypto.randomUUID()
sessionStorage.setItem('oauth_state', state)        // 记下这次登录请求的随机值
sessionStorage.setItem('back_to', location.pathname + location.search)
const p = new URLSearchParams({
response_type: 'code', client_id: CLIENT_ID, redirect_uri: REDIRECT_URI, state,
})
location.href = `${AUTH_SERVER}/authorize?${p}`
}

vue
<!-- src/views/OAuthCallback.vue -->
<script setup lang="ts">
import { onMounted, ref } from 'vue'
import { useRouter } from 'vue-router'
import http from '@/utils/request'
import { setToken } from '@/utils/auth'

const router = useRouter()
const error = ref('')

onMounted(async () => {
  const q = new URLSearchParams(location.search)
  const code = q.get('code'), state = q.get('state')
  const saved = sessionStorage.getItem('oauth_state')
  sessionStorage.removeItem('oauth_state')            // 一次性使用

  if (!code || !state || state !== saved) {           // 不是本浏览器发起的那次登录 → 不交出 code
    error.value = '登录回调无效，请重新登录'
    return
  }
  history.replaceState(null, '', location.pathname)   // 清掉地址栏里的 code

  const { token } = await http.post<any, { token: string }>('/auth/callback', { code }) // 后端换取凭据并签发 JWT（同事负责）
  setToken(token)
  router.replace(sessionStorage.getItem('back_to') || '/')
})
</script>
<template><div>{{ error || '登录中...' }}</div></template>

F3. 报工确认按钮：和后端 B6 对应
ts
// src/directives/permission.ts —— 没有权限就移除按钮（只是体验，后端还会再校验）
import type { Directive } from 'vue'
import { useUserStore } from '@/stores/user'
export const vPermission: Directive<HTMLElement, string> = {
mounted(el, { value }) {
if (!useUserStore().perms.includes(value)) el.remove()
},
}

vue
<script setup lang="ts">
import { ref } from 'vue'
import { ElMessage } from 'element-plus'
import http from '@/utils/request'
const props = defineProps<{ id: number }>()
const emit = defineEmits<{ done: [] }>()
const loading = ref(false)

async function confirm() {
  loading.value = true                       // 防止重复点击（只是体验，并发安全靠后端 SQL）
  try {
    await http.post(`/reports/${props.id}/confirm`)
    ElMessage.success('已确认')
    emit('done')                             // 通知父组件刷新列表
  } catch { /* 错误提示已在拦截器里弹出 */ }
  finally { loading.value = false }
}
</script>

<template>
  <el-button v-permission="'report:confirm'" :loading="loading" @click="confirm">确认</el-button>
</template>

F4. 公共表格组件（杰为风格，示意）
vue
<!-- ty-table-page.vue：分页、加载、请求都封装在内，业务页只需要传入接口 -->
<script setup lang="ts">
import { ref, reactive, onMounted } from 'vue'
const props = defineProps<{ api: (p: any) => Promise<{ list: any[]; total: number }>; query?: object }>()
const list = ref<any[]>([]), total = ref(0), loading = ref(false)
const page = reactive({ pageNo: 1, pageSize: 20 })

async function load() {
  loading.value = true
  try {
    const r = await props.api({ ...props.query, ...page })
    list.value = r.list; total.value = r.total
  } finally { loading.value = false }
}
onMounted(load)
defineExpose({ reload: load })
</script>

<template>
  <el-table v-loading="loading" :data="list"><slot /></el-table>
  <el-pagination v-model:current-page="page.pageNo" v-model:page-size="page.pageSize"
                 :total="total" @change="load" />
</template>

F5. 大数据子表单：VXE-Table 虚拟滚动
vue
<vxe-table :data="rows" height="500"
:scroll-y="{ enabled: true, gt: 50 }"
:edit-config="{ trigger: 'click', mode: 'cell' }">
  <!-- 只渲染可见区域的行；只有被激活的单元格才挂载输入框 -->
  <vxe-column field="qty" title="数量" :edit-render="{}">
    <template #edit="{ row }"><vxe-input v-model="row.qty" type="integer" /></template>
  </vxe-column>
</vxe-table>

ts
// 校验要基于数据，不能依赖 DOM：不可见的行根本没有 DOM
const errors = rows.value.filter(r => !(r.qty > 0))

F6. ECharts 实例复用
ts
let chart: echarts.ECharts | null = null
onMounted(() => { chart = echarts.init(el.value!) })
watch(data, d => chart?.setOption({ series: [{ type: 'bar', data: d }] }))  // 数据变化只更新配置，不重新 init
onBeforeUnmount(() => { chart?.dispose(); chart = null })                    // 组件卸载时释放实例

F7. 发版后提示刷新：version.json
ts
// 构建时注入版本号，并同时生成 dist/version.json，
// 例如 vite.config: define: { __APP_VERSION__: JSON.stringify(Date.now()) }
declare const __APP_VERSION__: string

const timer = setInterval(async () => {
const res = await fetch(`/version.json?t=${Date.now()}`, { cache: 'no-store' })  // 绕过浏览器缓存
const { version } = await res.json()
if (version !== __APP_VERSION__) {
clearInterval(timer)
ElMessageBox.confirm('系统已更新，是否刷新？').then(() => location.reload())
}
}, 60_000)

nginx
location = /version.json { add_header Cache-Control "no-store"; }

F8. wujie：PLM 作为主应用传登录态（实际方式【待确认】）
vue
<WujieVue name="mes" :url="MES_URL" :props="{ token: getToken(), user }" :sync="true" />

ts
// 子应用中读取
const { token } = (window as any).$wujie?.props || {}
