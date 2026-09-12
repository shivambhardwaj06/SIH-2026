# SwasthSetu — How we build the Complete Project

This README explains, in order, how to go from the architecture diagram to a real, working SwasthSetu kiosk. It covers only what was actually shown in the technical-approach diagram and what is confirmed to exist as of writing — where a step depends on getting access from a third party (UIDAI, ABDM, Bhashini), that is called out explicitly instead of assumed.

## Prototype ko kaise use karein (step-by-step) + aage kya real banega

1. **Welcome screen** — "EN / हिं" toggle try karo, phir "Touch to begin" dabao.
   - *Abhi:* sirf do languages ke sample text switch hote hain.
   - *Future me:* poora UI **Bhashini** ke through har Indian language me dynamically translate hoga, aur ek real voice-assist (Bhashini ASR/TTS) idle screen se hi sun/bol sakega.

2. **New ya Returning patient chuno.**
   - *Abhi:* sirf ek route select hota hai, koi verification nahi hoti.
   - *Future me:* ye choice **Layer 2 (Identity & Auth service)** ko call karegi jo decide karegi ki Aadhaar-based naya ABHA banana hai ya existing ABHA fetch karna hai.

3. **New patient → Aadhaar number + OTP daalo** (koi bhi 12 digit number aur 6 digit OTP chalega, ye fake hai).
   - *Abhi:* OTP verify karte hi ek random mock ABHA ID mil jaati hai.
   - *Future me:* ye **ABDM Sandbox ke enrolment API** se real call hogi (Aadhaar OTP part sandbox khud handle karta hai, tum direct UIDAI nahi chhoote), aur asli ABHA ID ban kar aayegi.

   **Returning patient → ABHA ID daalo ya QR scan karo (dummy).**
   - *Abhi:* koi bhi text daalne par mock profile load ho jaata hai.
   - *Future me:* **ABHA Sandbox Gateway** se patient ka real, existing profile aur pichli visit ka data fetch hoga.

4. **Mode chuno: Ayurvedic (AYUSH) ya Allopathic.**
   - *Abhi:* sirf ek color-coded route flag set hota hai.
   - *Future me:* ye flag decide karega ki **Layer 3 (Clinical Engine)** me kaunsa question-set aur kaunsi coding (NAMASTE ya SNOMED-CT) use hogi.

5. **Chief complaint bolo (mic dabao) ya type karo.**
   - *Abhi:* mic dabane par ek fixed sample sentence "transcribe" ho jaata hai.
   - *Future me:* **Bhashini ASR (ya Whisper)** real-time me patient ki actual awaaz ko text me convert karega, kisi bhi Indian language me.

6. **Assessment answer karo** (AYUSH ke liye Prakriti/Agni sliders, ya General ke liye severity/symptoms).
   - *Abhi:* answers se ek simple, hardcoded formula se result nikalta hai (jaise "Vata–Pitta").
   - *Future me:* ye **BioMistral/Med-Llama** model se real symptom-extraction hoga, aur AYUSH ka result ek clinician-approved **Dashavidha Pariksha rules engine** se aayega, coded to NAMASTE/SNOMED-CT.

7. **(Returning patient ho to) Follow-up question — better/same/worse.**
   - *Abhi:* sirf ek chip select hoti hai, koi comparison nahi hota.
   - *Future me:* ye pichli visit ke real stored data se compare hoga, jo Layer 4 ke cache/ABHA record se aaya hoga.

8. **Report scan karo ya existing reports dekho.**
   - *Abhi:* "scan" dabane par 1-2 second baad ek fixed fake result (jaise "Hemoglobin 11.2") dikh jaata hai.
   - *Future me:* real camera capture par **Tesseract ya AWS Textract** OCR chalega, aur "review existing" real cache/ABHA record se data laayega.

9. **Summary screen dekho.**
   - *Abhi:* sirf abhi tak diye gaye answers ek card me jod kar dikhaye jaate hain.
   - *Future me:* yahi summary ek proper **HL7 FHIR R4 bundle** ke roop me structure hogi, sync se pehle.

10. **"Sync & finish visit" dabao.**
    - *Abhi:* 4 progress lines animate hoti hain (fake delay ke saath), phir ek random Visit ID dikhta hai.
    - *Future me:* ye asli **ABDM M2 aur M3 APIs** ko call karega jo is visit ko patient ke central ABHA record se permanently link kar dega.

Right side ka **"System activity" log panel** har click par ek line likhta hai (CLIENT / AUTH / AI / OCR / SYNC tag ke saath) — ye batata hai ki us action ke peeche production me kaunsi cheez (Table wale kis layer) call hogi. Top bar ka **"What's simulated?"** button yahi list ek jagah dikha deta hai.

---

## Before you start: access you cannot skip

These are not libraries you `npm install` — they are external approvals. Apply for them on day one, because they take the longest.

| # | What | Where | Why you need it |
|---|------|-------|------------------|
| 1 | **ABDM Sandbox account** | `sandbox.abdm.gov.in` — fill the sandbox registration form | Gives you sandbox credentials for ABHA creation/verification (Milestone 1) and later M2/M3 data-push APIs. Approval typically takes a few days. |
| 2 | **UIDAI Aadhaar e-KYC / OTP access** | Only available to a licensed **AUA/KUA** (Authentication User Agency / KYC User Agency) under UIDAI | You almost certainly will **not** get direct production Aadhaar access as an individual or hackathon team. For a prototype/pilot, do Aadhaar-based ABHA creation through the **ABDM Sandbox's own enrolment flow**, which handles the Aadhaar OTP step on your behalf — you don't call UIDAI directly. |
| 3 | **Bhashini / ULCA account** | `bhashini.gov.in` → sign up → generate `userID` + `ulcaApiKey` (+ an inference API key) from "My Profile" | Needed for real speech-to-text and text-to-speech in Indian languages (replaces the prototype's canned transcript). |
| 4 | **NAMASTE Portal terminology access** | `namaste.ayush.gov.in` | This is where the standardized AYUSH morbidity codes (used to tag Ayurvedic diagnoses) are published. You need the code list/API access from here to code AYUSH findings consistently. |
| 5 | **SNOMED CT (India) access** | Distributed via the **National Resource Centre for EHR Standards (NRCeS), C-DAC Pune**, which also maintains the SNOMED CT AYUSH extension | Needed to code allopathic (and cross-walked AYUSH) findings to a standard terminology. |
| 6 | **AWS account** (only if you use Textract) | `aws.amazon.com` | Needed if you choose AWS Textract for OCR. Skip this entirely if you use Tesseract instead (see Step 4). |

Everything else below (frameworks, databases, servers) you control yourself and can start immediately, in parallel with these applications.

---

## Step 1 — Scaffold the project

Keep the four architecture layers as four separate, independently runnable pieces from day one — it will save you from a tangled monolith later.

```
swasthsetu/
  kiosk-app/       -> Layer 1: the touchscreen frontend (this is what swasthsetu-prototype.html stands in for)
  auth-service/    -> Layer 2: identity & auth
  clinical-engine/ -> Layer 3: AI triage / AYUSH assessment logic
  sync-service/    -> Layer 4: OCR + FHIR + ABDM sync
```

Each service is its own small backend that the kiosk app talks to over plain HTTP/JSON. This mirrors the four-layer diagram directly, so anyone new to the project can see which folder implements which box in the architecture.

---

## Step 2 — Build the kiosk frontend (Layer 1)

This is the part you already have a working reference for: `swasthsetu-prototype.html` shows exactly what every screen should look like and in what order.

1. Initialize a React project (e.g. with Vite) inside `kiosk-app/`.
2. Add Tailwind CSS for styling.
3. Rebuild each screen from the prototype as a React component: Welcome → Patient Type → Identity → Mode Select → Complaint → Assessment (AYUSH or General) → Follow-up (returning patients only) → Reports → Summary → Sync → Done.
4. Replace the prototype's mocked logic with real HTTP calls to the three backend services you'll build in Steps 3–5.
5. Once the web app runs correctly in a normal browser, wrap it with **Electron.js** to turn it into a locked-down kiosk application (full-screen, no browser chrome, no way to navigate away).
6. Wire in the **Bhashini ULCA API** for the mic button (speech-to-text) and for reading prompts/results aloud (text-to-speech), using the `userID`/`ulcaApiKey` from your Bhashini sign-up.

At the end of this step you have a real, running kiosk UI — it just doesn't verify identity, assess patients, or sync anywhere yet. That's Steps 3–5.

---

## Step 3 — Build the identity & auth service (Layer 2)

This service does three jobs: create an ABHA ID for new patients, verify an existing one for returning patients, and issue a session token the rest of the kiosk visit uses.

1. Stand up a small backend (Node.js/Express or Python/FastAPI — either is fine, pick whichever your team knows).
2. Integrate the **ABDM Sandbox's ABHA enrolment APIs** (Milestone 1) using the sandbox credentials from your registration. This flow handles Aadhaar-based ABHA creation for new patients without you touching UIDAI directly.
3. For returning patients, integrate the **ABHA verification / profile-fetch** endpoints from the same sandbox to pull their existing SwasthSetu profile.
4. Issue a short-lived session using **OAuth 2.0 + JWT** once identity is confirmed. Every later call from the kiosk (to the clinical engine and sync service) should carry this token.
5. Test this service entirely against the ABDM **sandbox** environment first — do not attempt production ABDM access until the sandbox flow works end to end and you've gone through ABDM's exit/production process.

---

## Step 4 — Build the clinical engine (Layer 3)

This service takes the patient's complaint and assessment answers and turns them into a coded clinical finding — differently for AYUSH and allopathic mode.

1. Stand up a Python service (FastAPI is a natural fit here, since most open medical NLP models are Python-first).
2. For symptom/complaint understanding, host an open medical language model such as **BioMistral** (a Mistral-based biomedical LLM available on Hugging Face) or a comparable **Med-Llama**-family model, and expose it behind a simple `/extract-symptoms` endpoint.
3. For AYUSH mode: implement the **Dashavidha Pariksha** as an explicit rules layer — a set of questions/inputs (Prakriti, Vikriti, Sara, Samhanan, Satmya, Satva, Ahara Shakti, Vyayama Shakti, Vaya, Pramana) that combine into a structured result, the same way the prototype's sliders and chips do, just with real clinical logic behind them instead of a placeholder calculation. Have an AYUSH clinician validate this rules layer — this is not something to design purely from documentation.
4. Code the AYUSH output against the **NAMASTE** terminology (Step 0, item 4) and the allopathic output against **SNOMED CT** (item 5), so both modes produce standard, machine-readable diagnosis codes rather than free text.
5. Return a structured JSON result (codes + a human-readable summary) to the kiosk frontend and onward to the sync service.

---

## Step 5 — Build the OCR & sync service (Layer 4)

This service digitizes paper reports and writes the finished visit to the patient's central ABHA record.

1. Stand up a backend for this service, with a local database — **MongoDB or PostgreSQL** — as the kiosk-side cache. This cache exists so a visit isn't lost if the internet connection to ABDM drops mid-consultation.
2. For OCR, choose one:
   - **Tesseract** (free, open-source, runs entirely on your own server — good starting point since it needs no external account), or
   - **AWS Textract** (a paid AWS service, generally more accurate on messy scans, needs the AWS account from Step 0).
3. Assemble the finished visit record as an **HL7 FHIR R4** bundle. There are open FHIR libraries in most languages (e.g. `fhir.resources` in Python, HAPI FHIR in Java) — use one rather than hand-building the JSON structure, since FHIR resources have many required fields.
4. Push that bundle through the **ABDM M2 and M3 APIs** (from your sandbox registration) to link it to the patient's central ABHA record.
5. Once the push succeeds, clear the local cache entry for that visit.

---

## Step 6 — Wire it all together

1. Point the kiosk frontend's API calls at your four real services instead of the mocked logic in the prototype.
2. Walk through the exact same journey the prototype demonstrates — patient type → identity → mode → complaint → assessment → follow-up → reports → summary → sync — but now each step is a real network call.
3. Use `swasthsetu-prototype.html` side-by-side as your acceptance test: for every screen, the real app should ask the same questions, in the same order, and end with the same outcome (a synced ABHA record) as the prototype simulates.
4. Test the **offline/poor-connectivity case** deliberately: disconnect the network mid-visit and confirm the kiosk-side cache (Step 5) holds the visit and retries the sync once connectivity returns, instead of losing the data.

---

## Step 7 — Before any real patient uses it

- Get sign-off that the deployment is reviewed against the **Digital Personal Data Protection Act, 2023**, since this handles real patients' health data.
- Confirm you've captured the **ABDM consent-manager** artifacts required before fetching or pushing a patient's health data — this is a formal part of the ABDM flow, not optional.
- Complete the ABDM **sandbox exit process** (their documented path from sandbox to production) before going live in a real clinic — you cannot point a live kiosk at the sandbox environment.
- Have a clinician review the Dashavidha Pariksha rules and the AI-suggested triage output before it's used to inform actual care decisions — none of the AI/rules components here should make an unreviewed clinical decision.

---

## Exact tech stack

This is the stack as specified in the architecture, with the concrete tools/services you'd use to implement each box.

| Layer | Component | Concrete tech |
|---|---|---|
| **1 · Client & Hardware** | Frontend framework | React.js |
| | Styling | Tailwind CSS |
| | Kiosk wrapper | Electron.js |
| | Voice input | Bhashini ASR (via ULCA API) or OpenAI Whisper |
| | Voice output | Bhashini TTS (via ULCA API) or ElevenLabs |
| **2 · Identity & Auth** | ABHA creation/verification | ABDM Sandbox APIs (Milestone 1), at `sandbox.abdm.gov.in` |
| | Aadhaar-based enrolment | Handled through the ABDM Sandbox enrolment flow (not direct UIDAI access, unless your organization is a licensed AUA/KUA) |
| | Returning-patient data fetch | ABHA Sandbox Gateway |
| | Session security | OAuth 2.0 + JWT |
| **3 · AI Triage & Clinical Engine** | Symptom/complaint extraction | BioMistral or a Med-Llama–family open medical LLM, served from a Python backend |
| | AYUSH coding | NAMASTE Portal terminology (`namaste.ayush.gov.in`) |
| | AYUSH assessment logic | Custom rules engine implementing the Dashavidha Pariksha, validated by a clinician |
| | Allopathic coding | SNOMED CT, distributed via NRCeS/C-DAC Pune |
| **4 · OCR & Central Sync** | Report digitization | Tesseract (open-source) or AWS Textract (paid) |
| | Clinical data format | HL7 FHIR R4 (use an existing FHIR library, don't hand-build the JSON) |
| | Kiosk-local cache | MongoDB or PostgreSQL |
| | Central record push | ABDM M2 and M3 APIs |
| **Cutting across all layers** | Backend language/framework | Node.js/Express or Python/FastAPI for the three backend services |
| | Version control & CI | Git, with a repo laid out as in Step 1 |

---

## What this README does **not** claim

To keep this accurate:
- It does not give you working API keys, endpoint URLs, or request/response schemas for ABDM, UIDAI, Bhashini, NAMASTE, or SNOMED CT — those are only available after you register with each program, and their exact API contracts should be read from each program's own current documentation at that time, not copied from here.
- It does not claim direct UIDAI production access is something you can obtain casually — it is restricted to licensed AUA/KUA entities, and the sandbox path is the realistic route for a prototype or pilot.
- It does not specify exact library version numbers, since these change over time and should be pinned against whatever is current when you actually start the project.
