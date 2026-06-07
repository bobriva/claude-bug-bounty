# Detection

Primary Technique:

Timing Attacks

---

CL.TE Detection

Transfer-Encoding: chunked

Short Content-Length

Backend waits for chunk.

Observable timeout.

---

TE.CL Detection

Chunked body

Long Content-Length

Backend waits for more data.

Observable timeout.

---

Differential Responses

Attack Request

Victim Request

Look for:

404

Unexpected Responses

Corrupted Methods