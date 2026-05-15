
  /btw 我可以另开一个会话，跑P01吗

    可以，但有 2 个注意点：

    1. 文件锁风险
    两个会话同时跑各自岗位互不冲突（P01 写 positions/P01_*/ 和 work/scratch/P01/，P04 写 positions/P04_*/ 和 work/scratch/P04/，路径完全隔离）。唯一共享的文件是 docs/positions-index.md，Step 8 时双方都会 Edit
    这一行。如果同时改最多只是 git diff 冲突，不会破坏数据。

    2. WebSearch / WebFetch / Agent 配额
    两个会话各自跑 exhibition-finder + 多个 verifier subagent，会并发消耗 web 工具配额和 token。如果遇到 rate limit 错误，是因为 P04 后台 4 个 verifier 已经在跑。可以先等 P04 vision finder 跑完（5 分钟），再开
     P01。

    操作步骤：
    # 新开终端，cd 到同一个 repo
    cd /mnt/e/category-based-rd-talent-source-mapping
    claude
    # 在新会话输入：
    /run-position P01

    P01 是植物基绝缘油岗位。展会方向：电力设备展（PW 前缀）+ 变压器油展 + 绿色能源展。新会话的 Claude 不知道我这边 P04 在跑，会独立走完整 8-step，包括闸门 1（问你审 exhibitions.csv）和闸门 2（dedup 决策）。

    建议：如果不急，等 P04 后台 4 个 verifier 跑完（~30 分钟）再开 P01，避免 web 工具并发竞争。如果急，现在开也不会破坏 P04 进度。
