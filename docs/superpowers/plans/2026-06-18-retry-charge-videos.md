# UP 投稿源充电视频重试 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 UP 投稿源卡片上增加“重试充电视频”按钮，把该投稿源下已识别的充电视频重新放回现有下载流程。

**Architecture:** 后端新增一个只支持 `submission` 源的 API，重置视频级分页下载状态和分页级视频内容下载状态，但不触发立即扫描。前端只在 UP 投稿源操作区展示按钮，调用该 API 后提示重置数量。

**Tech Stack:** Rust, Axum, SeaORM, Svelte, TypeScript, existing `runRequest`/toast utilities.

---

### Task 1: 后端状态重置 API

**Files:**
- Modify: `crates/bili_sync/src/api/response.rs`
- Modify: `crates/bili_sync/src/api/handler.rs`
- Modify: `crates/bili_sync/src/task/http_server.rs`

- [ ] **Step 1: Write the failing backend test**

Add tests near the existing video-source API tests in `crates/bili_sync/src/api/handler.rs`. The tests should call a new internal function named `retry_charge_videos_for_source_internal`.

```rust
#[tokio::test]
async fn retry_charge_videos_for_submission_resets_only_matching_charge_videos() {
    let db = setup_retry_charge_video_test_db().await;

    let response = retry_charge_videos_for_source_internal(db.clone(), "submission".to_string(), 1)
        .await
        .expect("应能重试投稿源充电视频");

    assert!(response.success);
    assert!(response.resetted);
    assert_eq!(response.resetted_videos_count, 1);
    assert_eq!(response.resetted_pages_count, 1);

    let charge_video = video::Entity::find_by_id(1).one(db.as_ref()).await.unwrap().unwrap();
    let normal_video = video::Entity::find_by_id(2).one(db.as_ref()).await.unwrap().unwrap();
    let other_charge_video = video::Entity::find_by_id(3).one(db.as_ref()).await.unwrap().unwrap();
    let charge_page = page::Entity::find_by_id(1).one(db.as_ref()).await.unwrap().unwrap();
    let normal_page = page::Entity::find_by_id(2).one(db.as_ref()).await.unwrap().unwrap();
    let other_charge_page = page::Entity::find_by_id(3).one(db.as_ref()).await.unwrap().unwrap();

    assert_eq!(VideoStatus::from(charge_video.download_status).get(VIDEO_STATUS_PAGE_DOWNLOAD_INDEX), 0);
    assert_eq!(PageStatus::from(charge_page.download_status).get(PAGE_STATUS_VIDEO_INDEX), 0);
    assert_eq!(VideoStatus::from(normal_video.download_status).get(VIDEO_STATUS_PAGE_DOWNLOAD_INDEX), STATUS_OK);
    assert_eq!(PageStatus::from(normal_page.download_status).get(PAGE_STATUS_VIDEO_INDEX), STATUS_OK);
    assert_eq!(VideoStatus::from(other_charge_video.download_status).get(VIDEO_STATUS_PAGE_DOWNLOAD_INDEX), STATUS_OK);
    assert_eq!(PageStatus::from(other_charge_page.download_status).get(PAGE_STATUS_VIDEO_INDEX), STATUS_OK);
}

#[tokio::test]
async fn retry_charge_videos_rejects_non_submission_sources() {
    let db = setup_retry_charge_video_test_db().await;

    let error = retry_charge_videos_for_source_internal(db, "favorite".to_string(), 1)
        .await
        .expect_err("非投稿源不应支持重试充电视频");

    assert!(error.to_string().contains("仅支持UP投稿源"));
}
```

- [ ] **Step 2: Run the backend tests and verify RED**

Run:

```powershell
cargo test -p bili_sync retry_charge_videos --lib
```

Expected: compile failure because `retry_charge_videos_for_source_internal`, `setup_retry_charge_video_test_db`, and `PAGE_STATUS_VIDEO_INDEX` do not exist yet.

- [ ] **Step 3: Implement the minimal backend code**

Add `RetryChargeVideosResponse` to `response.rs`.

```rust
#[derive(Serialize, ToSchema)]
pub struct RetryChargeVideosResponse {
    pub success: bool,
    pub source_id: i32,
    pub source_type: String,
    pub resetted: bool,
    pub resetted_videos_count: usize,
    pub resetted_pages_count: usize,
    pub message: String,
}
```

In `handler.rs`, add `PAGE_STATUS_VIDEO_INDEX = 1` locally if there is no exported constant, add the public route handler, and add an internal helper:

```rust
pub async fn retry_charge_videos_for_source(
    Extension(db): Extension<Arc<DatabaseConnection>>,
    Path((source_type, id)): Path<(String, i32)>,
) -> Result<ApiResponse<RetryChargeVideosResponse>, ApiError> {
    retry_charge_videos_for_source_internal(db, source_type, id)
        .await
        .map(ApiResponse::ok)
}
```

The internal helper should:

- return an API error unless `source_type == "submission"`;
- ensure the submission source exists;
- query `video` rows with `submission_id = id`, `valid = true`, `deleted = 0`, `auto_download = true`, `is_charge_video = true`;
- reset only `VIDEO_STATUS_PAGE_DOWNLOAD_INDEX` on matching videos;
- query all pages for matching video IDs and reset only `PAGE_STATUS_VIDEO_INDEX`;
- commit the transaction and call `notify_videos_changed()` if anything changed;
- not call `trigger_scan_now`, `resume_scanning`, or clear `next_scan_at`.

- [ ] **Step 4: Register the route and OpenAPI path**

Add `retry_charge_videos_for_source` to:

- the import list in `crates/bili_sync/src/task/http_server.rs`;
- the router as `POST /api/video-sources/{source_type}/{id}/retry-charge-videos`;
- the `#[openapi(paths(...))]` list in `handler.rs`.

- [ ] **Step 5: Run the backend tests and verify GREEN**

Run:

```powershell
cargo test -p bili_sync retry_charge_videos --lib
```

Expected: the new tests pass.

### Task 2: 前端 API 和 UP 投稿源按钮

**Files:**
- Modify: `web/src/lib/api.ts`
- Modify: `web/src/routes/video-sources/+page.svelte`

- [ ] **Step 1: Add the frontend API method**

In `web/src/lib/api.ts`, add a response type shape if no generated type exists, add a client method:

```ts
async retryChargeVideosForSource(
  sourceType: string,
  id: number
): Promise<ApiResponse<RetryChargeVideosResponse>> {
  return this.post<RetryChargeVideosResponse>(
    `/video-sources/${sourceType}/${id}/retry-charge-videos`
  );
}
```

Also expose it from the exported `api` object.

- [ ] **Step 2: Add source-local loading state and handler**

In `web/src/routes/video-sources/+page.svelte`, add a `Set<string>` tracking sources currently retrying. Add:

```ts
async function handleRetryChargeVideos(sourceId: number) {
  const key = `submission:${sourceId}`;
  if (retryingChargeVideoSources.has(key)) return;
  retryingChargeVideoSources = new Set(retryingChargeVideoSources).add(key);
  try {
    const response = await runRequest(() => api.retryChargeVideosForSource('submission', sourceId), {
      context: '重试充电视频失败'
    });
    if (!response) return;
    const data = response.data;
    if (data.resetted) {
      toast.success('已加入下一轮扫描', {
        description: `已重置 ${data.resetted_videos_count} 个视频、${data.resetted_pages_count} 个分页`
      });
    } else {
      toast.info('该 UP 投稿源没有可重试的充电视频');
    }
  } finally {
    const next = new Set(retryingChargeVideoSources);
    next.delete(key);
    retryingChargeVideoSources = next;
  }
}
```

- [ ] **Step 3: Render the button only for UP 投稿源**

Inside the existing `{#if sourceConfig.type === 'submission'}` action block, add a ghost icon button with `RotateCcwIcon` or a suitable existing lucide icon. Disable it while that source key is in the loading set. Do not render it for collection, favorite, watch_later, or bangumi.

- [ ] **Step 4: Run frontend verification**

Run:

```powershell
npm --prefix web run check
```

If the project has no `check` script, run:

```powershell
npm --prefix web run build
```

Expected: command exits 0.

### Task 3: Final verification and PR

**Files:**
- Verify all modified files.

- [ ] **Step 1: Run targeted Rust tests**

Run:

```powershell
cargo test -p bili_sync retry_charge_videos --lib
```

Expected: all targeted tests pass.

- [ ] **Step 2: Run formatting and frontend verification**

Run:

```powershell
cargo fmt --check
npm --prefix web run build
```

Expected: both commands exit 0.

- [ ] **Step 3: Inspect diff**

Run:

```powershell
git diff --stat
git diff --check
```

Expected: only planned files changed and no whitespace errors.

- [ ] **Step 4: Commit and open PR**

Stage only planned files, commit, push `feature/retry-charge-videos`, then create a PR to `main` with a concise Chinese summary and verification notes.
