# 部署说明 · 阿里云 OSS 静态网站托管

本项目从 Netlify 迁到了阿里云 OSS。工作流在 [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml)，
**push 到 `main` 就自动部署**，和以前 Netlify 的体验一致。

项目是纯静态站点（`index.html` + `shenshitang.html`，无 `package.json`、无构建步骤），
所有图片/视频/音效都以 base64 内嵌在 HTML 里，资源引用全部是相对路径，
所以放在 Bucket 根目录或任意子路径下都能正常运行。

---

## 一、GitHub 侧：加两个密钥

仓库 → **Settings → Secrets and variables → Actions → New repository secret**，添加：

| Name | Value |
|---|---|
| `ALIYUN_ACCESS_KEY_ID` | RAM 子账号的 AccessKey ID |
| `ALIYUN_ACCESS_KEY_SECRET` | RAM 子账号的 AccessKey Secret |

名字必须一字不差，工作流按这两个名字读。AccessKey 怎么来见下面第三步。

## 二、GitHub 侧：改工作流里的两个占位符

打开 `.github/workflows/deploy.yml`，改 `env:` 段：

```yaml
OSS_ENDPOINT: oss-cn-hangzhou.aliyuncs.com   # ← 换成你 Bucket 所在地域的 Endpoint
OSS_BUCKET: YOUR-BUCKET-NAME                  # ← 换成你的 Bucket 名
```

两个值都在**阿里云控制台 → 对象存储 OSS → 选中你的 Bucket → 「概览」页**：

- **Bucket 名称**：概览页最上方。
- **Endpoint**：概览页「访问端口」区域，取**外网访问 Endpoint**，
  形如 `oss-cn-hangzhou.aliyuncs.com`。
  ⚠️ 只填域名：不要带 `https://`，也不要用那个带 Bucket 名的完整域名
  （`your-bucket.oss-cn-hangzhou.aliyuncs.com` 是错的）。

## 三、阿里云侧：一次性配置

1. **创建 Bucket**
   控制台 → 对象存储 OSS → Bucket 列表 → 创建 Bucket。
   - 地域：选离用户近的（国内一般 华东1·杭州 / 华北2·北京）。
   - **读写权限：公共读**（`public-read`）。私有权限下网页打不开。

2. **开启静态网站托管**
   进入 Bucket → 左侧 **数据管理 → 静态页面**（部分版本叫「基础设置 → 静态页面」）：
   - **默认首页**：`index.html`
   - **默认 404 页**：留空，或也填 `index.html`
   - 保存。

3. **建 RAM 子账号拿 AccessKey**（不要用主账号 AccessKey）
   控制台 → **访问控制 RAM → 身份管理 → 用户 → 创建用户**：
   - 勾选 **「使用永久 AccessKey 访问」**（OpenAPI 调用访问）。
   - 创建后**立刻抄下 AccessKey Secret**，页面关掉就再也看不到了。
   - 给这个用户授权：**权限管理 → 新增授权 → `AliyunOSSFullAccess`**。
     想收紧的话，可以自定义策略只授权这一个 Bucket：

     ```json
     {
       "Version": "1",
       "Statement": [
         {
           "Effect": "Allow",
           "Action": ["oss:PutObject", "oss:GetObject", "oss:ListObjects", "oss:DeleteObject"],
           "Resource": ["acs:oss:*:*:YOUR-BUCKET-NAME", "acs:oss:*:*:YOUR-BUCKET-NAME/*"]
         }
       ]
     }
     ```
   - 把 AccessKey ID / Secret 填进第一步的 GitHub Secrets。

## 四、⚠️ 必须绑自定义域名，否则网页会被"下载"而不是打开

这是 OSS 和 Netlify 最大的区别，务必先知道：

**用 Bucket 默认域名（`*.oss-cn-xxx.aliyuncs.com`）访问 HTML 文件时，
阿里云会强制加上 `Content-Disposition: attachment` 响应头，浏览器直接弹下载框，
不会渲染网页。** 静态网站托管的 `*.oss-website-cn-xxx.aliyuncs.com` 域名同样受限。

也就是说：**不绑自定义域名，这个站就没法在线玩。**

绑定方式：Bucket → **传输管理 → 域名管理 → 绑定域名**，然后在你的 DNS 服务商
把该域名 CNAME 到 Bucket 的外网 Endpoint。

**国内节点的 Bucket 绑定自定义域名，域名必须完成 ICP 备案**（阿里云会校验），
备案一般要 1–20 个工作日。想跳过备案，只能把 Bucket 建在
**中国香港或海外地域**（那边不需要备案，但国内访问速度会慢一些）。

如果之后要接 CDN 加速：控制台 → CDN → 添加域名 → 源站类型选 OSS 域名。
**国内 CDN 节点同样强制要求 ICP 备案。**

## 五、日常使用

改完代码 `git push` 到 `main` 就完事了。想看部署进度：仓库 **Actions** 标签页。
也可以在 Actions 页面点 **Run workflow** 手动触发一次（工作流开了 `workflow_dispatch`）。

### 两个已知的小脾气

- **`-u` 增量在 CI 里基本不起作用。**
  `ossutil cp -u` 靠比对文件修改时间决定要不要传，而 `actions/checkout` 每次都会把
  文件时间戳刷成"现在"，所以实际每次都是全量上传。本项目一共才 3MB，无所谓；
  以后文件多了想真增量，可以改用 `ossutil sync`。

- **`cp` 不会删除线上多余的文件。**
  你在仓库里删掉一个文件，OSS 上那份还留着。要让线上和仓库严格一致，
  把上传那步换成：

  ```bash
  ossutil sync "$SITE_DIR"/ oss://$OSS_BUCKET/ --delete -f
  ```

  ⚠️ `--delete` 会删掉 Bucket 里所有不在本次上传范围内的对象，
  如果这个 Bucket 还存着别的东西，别用。

## 六、从 Netlify 迁走还要手动做的事

见仓库 README「部署」一节，或直接照下面做：

1. **停掉 Netlify 自动构建**
   Netlify 后台 → 选中站点 → **Site configuration → Build & deploy → 
   Continuous deployment → Stop builds**。
   或者干脆删站：**Site configuration → General → 拉到最底 → Delete this site**。

2. **删掉 GitHub 上残留的 Netlify webhook**
   仓库 → **Settings → Webhooks**，找 payload URL 里带 `netlify.com` 的那条
   （通常是 `https://api.netlify.com/hooks/github`），**Delete**。
   不删的话每次 push 还会去戳 Netlify 一下，虽然构建停了但没必要留着。

3. 顺便检查 **Settings → Integrations / GitHub Apps**，
   如果装了 Netlify 的 GitHub App，一并撤销对本仓库的授权。
