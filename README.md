# Attendance AI Service

A standalone AI microservice for employee face verification.

The service is responsible only for:

- Face detection
- Liveness / anti-spoofing
- Face embedding generation
- Face embedding comparison
- Returning AI decisions as JSON

It does **not** manage employees, attendance records, or databases.

---

## 1. AI Architecture

```text
                    AI SERVICE
                       │
          ┌────────────┴────────────┐
          │                         │
       Enrollment                Verification
          │                         │
          ▼                         ▼
    Face Detection             Face Detection
          │                         │
          ▼                         ▼
   Face Embedding               Liveness
          │                         │
          ▼                         ▼
   Return embedding          Temporal Liveness
      as JSON                       │
                                    ▼
                              Face Embedding
                                    │
                                    ▼
                             Compare Embeddings
                                    │
                                    ▼
                             Verification Result
```

The AI service is stateless with respect to user data.

Incoming frames, face crops, and temporary embeddings are processed in memory and are not persisted.

---

## 2. Models

### Face Detection — YuNet

Used to:

- Detect faces
- Return face bounding boxes
- Reject frames with no face
- Reject frames containing multiple faces

```text
Input image
    ↓
YuNet
    ↓
Face bounding box + confidence
```

### Liveness — MiniFASNet Ensemble

Two models are used:

```text
2.7_80x80_MiniFASNetV2
4_0_0_80x80_MiniFASNetV1SE
```

For every detected face:

```text
Face bounding box
       │
       ├── 2.7× crop → MiniFASNetV2
       │
       └── 4.0× crop → MiniFASNetV1SE
                    │
                    ▼
                Ensemble
                    │
                    ▼
              Live / Spoof
```

The liveness crop and model input are kept in memory.

### Face Recognition

The recognition component generates a **128-dimensional face embedding**.

```text
Face
 ↓
FaceRecognizer
 ↓
128-D embedding
```

The exact recognition model is isolated behind `FaceRecognizer`, so it can be replaced without changing the API or verification pipeline.

---

# 3. Enrollment Pipeline

Endpoint:

```http
POST /api/v1/enroll
```

Current flow:

```text
Enrollment image
       ↓
Image decoding
       ↓
YuNet
       ↓
Exactly one face
       ↓
Face recognition model
       ↓
128-D embedding
       ↓
JSON response
```

Example response:

```json
{
  "success": true,
  "embedding": [
    0.0123,
    -0.0312
  ]
}
```

The full embedding contains 128 floating-point values.

The AI service does not store the embedding permanently.

The caller is responsible for storing it.

---

# 4. Verification Pipeline

Endpoint:

```http
POST /api/v1/verify
```

The caller provides:

```text
employee_id
enrolled_embedding
camera frames
```

Pipeline:

```text
Camera frames
      ↓
Process every frame
      ↓
YuNet
      ↓
Exactly one face?
      │
      ├── No face → invalid frame
      ├── Multiple faces → invalid frame
      └── One face
             ↓
          Liveness
             ↓
     MiniFASNet ensemble
             ↓
      Liveness score/frame
             ↓
     Temporal aggregation
             ↓
        Live?
         │
         ├── No → reject
         │
         └── Yes
                ↓
       Select best valid frame
                ↓
         Face embedding
                ↓
          Probe embedding
                ↓
       Compare with enrolled
          128-D embedding
                ↓
        Identity match?
                ↓
        verified / rejected
```

Conceptually:

```text
stored_embedding = enrollment embedding
probe_embedding   = embedding from current verification frame
```

Then:

```text
similarity = compare(probe_embedding, stored_embedding)
```

---

# 5. Temporal Liveness

During verification, multiple frames are sent.

Target flow:

```text
Frame 1
Frame 2
Frame 3
Frame 4
Frame 5
...
        ↓
Liveness for each frame
        ↓
Temporal aggregation
        ↓
Final passive liveness decision
```

Current temporal policy:

```text
Minimum valid frames: 3
Minimum live ratio:   70%
```

Example:

```text
5 valid frames
4 live
1 spoof

live ratio = 4 / 5 = 80%

→ liveness passes
```

Example:

```text
5 valid frames
2 live
3 spoof

live ratio = 2 / 5 = 40%

→ liveness fails
```

---

# 6. API Overview

| API | Purpose |
|---|---|
| `GET /api/v1/health` | Service health |
| `POST /api/v1/detect` | Test face detection |
| `POST /api/v1/liveness` | Test passive liveness |
| `POST /api/v1/enroll` | Generate enrollment embedding |
| `POST /api/v1/verify` | Liveness + embedding comparison |

---

# 7. Health API

```http
GET /api/v1/health
```

Example:

```json
{
  "status": "ok",
  "version": "0.1.0"
}
```

---

# 8. Detection API

```http
POST /api/v1/detect
```

Input:

```text
image
```

Purpose:

Test YuNet independently.

---

# 9. Liveness API

```http
POST /api/v1/liveness
```

Input:

```text
files[] = camera frames
```

Pipeline:

```text
Frames
  ↓
YuNet
  ↓
MiniFASNetV2 + MiniFASNetV1SE
  ↓
Frame-level liveness scores
  ↓
Temporal aggregation
  ↓
LIVE / SPOOF
```

The response includes diagnostics such as:

```text
total_frames
valid_frames
live_frames
spoof_frames
liveness_score
per-frame results
```

---

# 10. Enrollment API

```http
POST /api/v1/enroll
```

Current request:

```text
employee_id
file
```

The AI service generates a face embedding and returns it as JSON.

The AI service does not create or access an employee database.

---

# 11. Verification API

```http
POST /api/v1/verify
```

Current request:

```text
employee_id

enrolled_embedding=<JSON string containing the 128-D embedding>

files[]=frame_1
files[]=frame_2
files[]=frame_3
files[]=frame_4
files[]=frame_5
```

The AI service processes the frames, performs liveness, creates a probe embedding, compares it with the enrolled embedding, and returns the verification result.

---

# 12. Swagger Testing

Start the service:

```bash
docker compose up --build
```

Open:

```text
http://localhost:8000/docs
```

Recommended order:

```text
1. GET  /api/v1/health
2. POST /api/v1/detect
3. POST /api/v1/liveness
4. POST /api/v1/enroll
5. POST /api/v1/verify
```

For liveness testing use real camera captures, printed photos, and replayed videos.

---

# 13. UI Testing Without `ui.py`

`app/api/ui.py` is only a development convenience and is not required by the AI API.

The production API can run without registering it in `app/main.py`.

it's a  separate development UI you can test the pipeline :

Open:

```text
http://localhost:8000
```

# 14. Data Handling

The AI service should not persist user biometric input.

Do not save:

```text
Camera frames
Face crops
Temporary images
Temporary embeddings
```

Face crops are temporary inference artifacts and should remain in memory only.

---

# 15. Production Verification Decision

The final AI decision is conceptually:

```text
ACCEPT =
    exactly_one_face
    AND
    liveness_passed
    AND
    identity_match
```

Otherwise:

```text
REJECT
```

---

# 16. Project Structure

```text
attendance-ai/
│
├── app/
│   ├── main.py
│   │
│   ├── api/
│   │   ├── health.py
│   │   ├── detect.py
│   │   ├── liveness.py
│   │   ├── enroll.py
│   │   └── verify.py
|   |   └── ui.py
│   │
|   ├── core/
│   │   ├── config.py    
|   |
│   ├── face/
│   │   ├── detector.py
│   │   └── recognizer.py
│   │
│   ├── liveness/
│   │   ├── ensemble.py
│   │   └── temporal.py
│   │
│   ├── pipeline/
│   │   └── enrollment.py
│   │
│   └── schemas/
|       └── response.py
|       └── request.py
│
├── models/
│   ├── detection/
│   ├── liveness/
│   └── recognition/
│
├── scripts/
├── tests/
│
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
└── README.md
```

---

# 17. Docker

Build:

```bash
docker compose build
```

Run:

```bash
docker compose up
```



# 18. Resource Limits

The current target server has approximately:

```text
2 vCPU
8 GB RAM
```

A starting container limit can be:

```yaml
services:
  attendance-ai:
    build: .
    ports:
      - "8000:8000"
    cpus: "1.5"
    mem_limit: 4g
    restart: unless-stopped
```

The final values should be validated with load testing.

---

# 19. Testing Strategy

Test each AI stage independently:

```text
Health
  ↓
Face Detection
  ↓
Liveness
  ↓
Enrollment Embedding
  ↓
Verification
```

Liveness test cases:

```text
Real person
Printed photo
Phone screen replay
Laptop replay video
```

Recognition test cases:

```text
Correct person
Wrong person
```

Measure false acceptance and false rejection on the target camera/device before production.

---

# 20. Summary

The AI project is an independent inference service:

```text
Input image / frames
        ↓
Face Detection
        ↓
Liveness
        ↓
Face Embedding
        ↓
Embedding Comparison
        ↓
JSON AI Decision
```

Enrollment:

```text
Image → Detection → Embedding → JSON
```

Verification:

```text
Frames + enrolled embedding
        ↓
Detection
        ↓
Liveness
        ↓
Probe embedding
        ↓
Comparison
        ↓
Verified / Rejected
```

The AI service remains independent, database-free, and focused only on face AI inference.
