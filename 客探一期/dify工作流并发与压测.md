你这个问题可以拆成两块：`看并发` 和 `做压测`。  
先给可落地方案。

## 怎么看并发
1. 看单次 Workflow 内部是否并行执行：在 Studio 的 `Run History -> Tracing` 看节点顺序和每个节点耗时，能直接看出并行分支有没有真正并发起来。  
2. 看线上请求表现：在 `Monitor -> Logs` 看每次交互的响应时间、token、错误。  
3. 看“当前同时在跑多少个 workflow 实例”（最实用）：调用 Workflow Logs API，按 `status=running` 过滤，读返回里的 `total`。这基本就是当前并发实例数。  
4. 看系统并发上限（自部署）：重点看 `.env` 里的这些参数：  
`APP_MAX_ACTIVE_REQUESTS`、`SERVER_WORKER_AMOUNT`、`SERVER_WORKER_CONNECTIONS`、`CELERY_WORKER_AMOUNT` / `CELERY_AUTO_SCALE` / `CELERY_MAX_WORKERS` / `CELERY_MIN_WORKERS`、`GRAPH_ENGINE_MIN_WORKERS` / `GRAPH_ENGINE_MAX_WORKERS`、`MAX_PARALLEL_LIMIT`。

PowerShell 实时看 running 数（每秒刷新）：

```powershell
$base = "https://your-dify-api/v1"
$key  = "app-xxxx"

while ($true) {
  $r = Invoke-RestMethod -Method Get `
    -Uri "$base/workflows/logs?status=running&page=1&limit=1" `
    -Headers @{ Authorization = "Bearer $key" }

  "{0:HH:mm:ss} running={1}" -f (Get-Date), $r.total
  Start-Sleep -Seconds 1
}
```

## 怎么做压力测试（推荐 k6）
1. 固定一个稳定测试输入，先做单用户基线（1 RPS，3-5 分钟）。  
2. 做阶梯压测（5 -> 10 -> 20 RPS），观察 p95 延迟、错误率、running 数。  
3. 做极限压测（持续提速到错误率明显上升）。  
4. 做稳态压测（在“极限吞吐的 60%-70%”跑 30-60 分钟）。  
5. 每次只改一个变量（比如只改 `CELERY_MAX_WORKERS`），避免混淆结论。

`dify_workflow_load.js` 示例（可直接跑）：

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Counter, Trend } from 'k6/metrics';

const bizFailed = new Counter('biz_failed');
const e2e = new Trend('workflow_e2e_ms');

export const options = {
  scenarios: {
    ramp: {
      executor: 'ramping-arrival-rate',
      timeUnit: '1s',
      startRate: 1,
      preAllocatedVUs: 20,
      maxVUs: 200,
      stages: [
        { target: 5, duration: '2m' },
        { target: 10, duration: '3m' },
        { target: 20, duration: '3m' },
        { target: 0, duration: '1m' }
      ]
    }
  },
  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<5000'],
    workflow_e2e_ms: ['p(95)<15000']
  }
};

const BASE_URL = __ENV.BASE_URL; // 例如: https://your-dify-api/v1
const API_KEY = __ENV.API_KEY;
const HEADERS = {
  Authorization: `Bearer ${API_KEY}`,
  'Content-Type': 'application/json'
};

export default function () {
  const start = Date.now();

  const payload = JSON.stringify({
    inputs: {
      user_query: __ENV.QUERY || 'health check'
    },
    response_mode: 'blocking',
    user: `loadtest-${__VU}-${__ITER}`
  });

  const res = http.post(`${BASE_URL}/workflows/run`, payload, {
    headers: HEADERS,
    timeout: '120s'
  });

  if (!check(res, { 'run 200': r => r.status === 200 })) return;

  let body = {};
  try {
    body = res.json();
  } catch (e) {
    bizFailed.add(1);
    return;
  }

  let status = body?.data?.status;
  const runId = body?.workflow_run_id || body?.data?.id;

  if (status === 'running' && runId) {
    for (let i = 0; i < 120; i++) {
      sleep(1);
      const poll = http.get(`${BASE_URL}/workflows/run/${runId}`, { headers: HEADERS });
      if (poll.status !== 200) continue;
      status = poll.json()?.status;
      if (status !== 'running') break;
    }
  }

  if (status && status !== 'succeeded') bizFailed.add(1);
  e2e.add(Date.now() - start);
}
```

运行：

```bash
k6 run -e BASE_URL=https://your-dify-api/v1 -e API_KEY=app-xxxx dify_workflow_load.js
```

## 两个关键坑
1. Dify Cloud 的 `blocking` 模式有 Cloudflare 100 秒超时，长流程压测容易被这个限制干扰。  
2. 如果 Workflow 用了知识库，Dify Cloud 有知识库请求频率限制（按套餐不同），压测时会先撞这个限额，不一定是你机器顶不住。

## 补充判断（基于文档信息推断）
`Dashboard` 文档列的是消息量/活跃用户/交互深度/token，用来做趋势分析；没有专门“实时并发曲线”字段，所以实时并发建议走 `workflows/logs?status=running` 或外部观测（Langfuse/LangSmith/Grafana）。

## Sources
- https://docs.dify.ai/api-reference/workflow-execution/execute-workflow
- https://docs.dify.ai/api-reference/workflow-execution/get-workflow-run-detail
- https://docs.dify.ai/api-reference/workflow-execution/get-workflow-logs
- https://docs.dify.ai/en/use-dify/debug/history-and-logs
- https://docs.dify.ai/en/use-dify/monitor/logs
- https://docs.dify.ai/en/use-dify/monitor/analysis
- https://raw.githubusercontent.com/langgenius/dify/main/docker/.env.example
- https://docs.dify.ai/en/use-dify/knowledge/knowledge-request-rate-limit
- https://docs.dify.ai/en/use-dify/workspace/model-providers#configure-model-load-balancing

如果你愿意，我可以下一步按你当前机器配置，给你一版“目标并发 N、预计 token 成本、压测时长”的具体参数表。
