# Article Writer with Handoffs

A multi-agent pipeline that researches a topic, generates an image, selects the best writer style, and produces a finished article in markdown.

## Pipeline overview

```
User prompt
    → Orchestrator Agent
        → research_agent tool  ×2  (pick best brief)
        → image_agent tool         (generate & upload image)
        → handoff to journalist / educator / advisor
            → final article (markdown)
```

---

## 1. `fetch_url` — why trafilatura alone is not enough

**trafilatura** is excellent at extracting readable text from ordinary web pages — it strips navigation, ads, and boilerplate and returns the main article body.

The problem is that many high-quality sources are **not web pages**:
- Academic papers are distributed as **PDF** files
- Government reports, policy documents, and press kits are often **Word (.docx)** files

trafilatura does not handle binary file formats. If you pass a PDF URL to `trafilatura.fetch_url()`, it will either return nothing or return garbled binary text.

### How it is solved here

`fetch_url` runs a lightweight **content detection step first**:

**Step 1 — HEAD request.**
Before downloading the full file, send a `HEAD` request and read the `Content-Type` header:
```
application/pdf              → is_pdf = True
application/vnd.openxmlformats-officedocument.wordprocessingml.document → is_docx = True
```

**Step 2 — URL extension fallback.**
If the server doesn't return a useful `Content-Type` (some servers don't), check whether the URL path ends in `.pdf` or `.docx`.

**Step 3 — branch by file type:**
- **PDF** → download the full file, pass bytes to `pypdf.PdfReader`, extract text page by page
- **DOCX** → download the full file, pass bytes to `python-docx`, extract text paragraph by paragraph
- **Everything else** → pass to `trafilatura` as normal

This way the research agent can follow any URL a search result returns, regardless of format.

### Dependencies

```bash
pip install pypdf python-docx trafilatura
```

---

## 2. Image generation — why the image must be uploaded to R2

**DALL-E 3** returns a temporary CDN URL (valid for ~1 hour). You can put that URL directly in a markdown `![image](url)` tag and it renders immediately.

**`gpt-image-1`** (the newer GPT image model) does **not** return a URL at all. It returns the image as a **base64-encoded string** in the response body. There is no hosted URL — you get raw bytes, period.

This means:
- You can save the bytes to a local file, but local file paths don't work in markdown rendered in a browser or shared anywhere
- You need somewhere to host the image and get back a stable public URL

### How it is solved here — Cloudflare R2

**Step 1 — Generate the image.**
Call `openai.images.generate()` with `model="gpt-image-1-mini"`. Read `response.data[0].b64_json` and decode to bytes.

**Step 2 — Upload to R2.**
Use `boto3` (the AWS S3 client — R2 is S3-compatible) to upload the bytes to a Cloudflare R2 bucket:
```python
s3 = boto3.client(
    "s3",
    endpoint_url=f"https://{R2_ACCOUNT_ID}.r2.cloudflarestorage.com",
    aws_access_key_id=R2_ACCESS_KEY,
    aws_secret_access_key=R2_SECRET_KEY,
    region_name="auto",
)
s3.put_object(Bucket=R2_BUCKET, Key=key, Body=img_bytes, ContentType="image/png")
```

**Step 3 — Construct a public URL.**
If the R2 bucket has public access enabled (or you have a custom domain pointed at it):
```python
url = f"{R2_PUBLIC_URL}/{key}"   # e.g. https://pub-abc123.r2.dev/uuid.png
```

**Step 4 — Return the URL.**
`generate_image` returns this URL string. The image agent passes it to the orchestrator, which passes it to the writer. The writer embeds it as `![Hero Image](https://pub-abc123.r2.dev/uuid.png)` and it renders everywhere — in the notebook, in a browser, in a shared document.

### Required environment variables (`.env`)

```
R2_ACCOUNT_ID=your_cloudflare_account_id
R2_BUCKET=your_bucket_name
R2_ACCESS_KEY=your_r2_access_key_id
R2_SECRET_KEY=your_r2_secret_access_key
R2_PUBLIC_URL=https://pub-xxxx.r2.dev   # or your custom domain
```

### Dependencies

```bash
pip install boto3
```

---

## 3. Handoffs — why `WriterRank` and `WriterSelectionInfo` are needed

By default, when the orchestrator calls `handoff(journalist_agent)`, the SDK fires the handoff and the writer starts writing. You have no visibility into **why** that writer was chosen over the others.

If you want to observe the orchestrator's reasoning — which is important for debugging and for understanding the model's behaviour — you need to capture that reasoning as **structured data** at the moment the handoff decision is made.

### The mechanism: `input_type`

The `handoff()` function accepts an `input_type` parameter. When you supply a Pydantic model as `input_type`, the SDK:
1. Generates a JSON schema from the model and includes it in the tool description sent to the orchestrator
2. Forces the orchestrator to produce a valid JSON object matching that schema as the handoff input
3. Deserialises that JSON into an instance of your model and passes it to your `on_handoff` callback

This gives you a typed Python object with the orchestrator's reasoning in it, before the writer agent runs.

### Step 1 — Define `WriterRank` for a single writer's entry

```python
class WriterRank(BaseModel):
    writer: str = Field(description="Writer name: journalist, educator, or advisor")
    rank: int   = Field(description="Rank: 1 = best fit, 2 = second, 3 = third")
    reason: str = Field(description="Why this writer was ranked here")
```

`Field(description=...)` is important — these descriptions are injected into the JSON schema and are the only way the model knows what each field means.

### Step 2 — Define `WriterSelectionInfo` for the full decision

```python
class WriterSelectionInfo(BaseModel):
    which_writer:    str             = Field(description="Name of the chosen writer: journalist, educator, or advisor")
    reason:          str             = Field(description="Why this writer is the best fit for the topic")
    writers_ranking: list[WriterRank] = Field(description="All three writers ranked from best to worst fit")
```

This captures:
- the final choice
- the primary reason for it
- the full ranking of all three writers with individual reasons

### Step 3 — Define `on_handoff` to print the decision

```python
async def on_handoff(_ctx, decision: WriterSelectionInfo):
    ranked = sorted(decision.writers_ranking, key=lambda w: w.rank)
    ranking_str = ", ".join([f"#{r.rank} {r.writer}: {r.reason}" for r in ranked])
    print(f"✍️  Selected: {decision.which_writer}")
    print(f"   Reason:   {decision.reason}")
    print(f"   Rankings: {ranking_str}")
```

### Step 4 — Pass `input_type` to each handoff

```python
orchestrator_agent = Agent(
    ...
    handoffs=[
        handoff(journalist_agent, on_handoff=on_handoff, input_type=WriterSelectionInfo),
        handoff(educator_agent,   on_handoff=on_handoff, input_type=WriterSelectionInfo),
        handoff(advisor_agent,    on_handoff=on_handoff, input_type=WriterSelectionInfo),
    ]
)
```

Every handoff uses the same `input_type` and the same `on_handoff` callback. Whichever writer the orchestrator picks, you will see a printed line like:

```
✍️  Selected: educator
   Reason:   Topic is foundational science requiring zero assumed knowledge
   Rankings: #1 educator: best for beginners, #2 journalist: too niche, #3 advisor: no action angle
```
