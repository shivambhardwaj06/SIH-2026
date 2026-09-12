# SwasthSetu — How We Build the Complete Project

This README explains, step by step, how we can go from our architecture diagram to a real, working SwasthSetu kiosk. It covers what we have shown in the technical-approach diagram and what is confirmed to exist as of writing. Wherever a step depends on getting access from a third party (UIDAI, ABDM, Bhashini), we have called that out explicitly instead of assuming it.

## How We Use the Prototype (Step-by-Step) + What We Will Build Next

1. **Welcome screen** — We can try the "EN / हिंदी" toggle and then press "Touch to begin".

   * *Right now:* We are only switching between sample text in two languages.
   * *Future:* We will make the complete UI dynamically translate into every Indian language through **Bhashini**, and we will add real voice assistance (Bhashini ASR/TTS) so the kiosk can listen and speak from the idle screen itself.

2. **We choose New or Returning patient.**

   * *Right now:* We only select a route; no verification is performed.
   * *Future:* This choice will call our **Layer 2 (Identity & Auth service)**, which will decide whether we need to create a new ABHA using Aadhaar or fetch an existing ABHA.

3. **New patient → We enter Aadhaar number + OTP** (any 12-digit number and 6-digit OTP works here because this is a fake prototype).

   * *Right now:* Once we verify the OTP, we receive a random mock ABHA ID.
   * *Future:* We will make the real call through the **ABDM Sandbox enrolment API**. The Aadhaar OTP part is handled by the sandbox itself, so we do not directly interact with UIDAI, and the actual ABHA ID will be generated.

   **Returning patient → We enter the ABHA ID or scan the QR code (dummy).**

   * *Right now:* Entering any text loads a mock profile.
   * *Future:* We will fetch the patient's real existing profile and previous-visit data through the **ABHA Sandbox Gateway**.

4. **We choose the mode: Ayurvedic (AYUSH) or Allopathic.**

   * *Right now:* We only set a color-coded route flag.
   * *Future:* This flag will decide which question set and which coding system (NAMASTE or SNOMED-CT) we use in **Layer 3 (Clinical Engine)**.

5. **We speak our chief complaint using the mic or type it.**

   * *Right now:* Pressing the mic converts a fixed sample sentence into a "transcription".
   * *Future:* **Bhashini ASR (or Whisper)** will convert the patient's actual voice into text in real time, in any Indian language.

6. **We complete the assessment** (Prakriti/Agni sliders for AYUSH, or severity/symptoms for General).

   * *Right now:* We get a result from a simple hardcoded formula, such as "Vata–Pitta".
   * *Future:* We will use the **BioMistral/Med-Llama** model for real symptom extraction, while the AYUSH result will come from a clinician-approved **Dashavidha Pariksha rules engine**, coded to NAMASTE/SNOMED-CT.

7. **For a returning patient → We answer the follow-up question — better/same/worse.**

   * *Right now:* We only select one chip; there is no comparison.
   * *Future:* We will compare this with the real stored data from the previous visit, retrieved from the Layer 4 cache/ABHA record.

8. **We scan a report or review existing reports.**

   * *Right now:* When we press "scan", a fixed fake result, such as "Hemoglobin 11.2", appears after a 1–2 second delay.
   * *Future:* We will run **Tesseract or AWS Textract** OCR on the actual camera capture, while "review existing" will fetch data from the real cache/ABHA record.

9. **We review the Summary screen.**

   * *Right now:* We combine all the answers provided so far into one card.
   * *Future:* We will structure this summary as a proper **HL7 FHIR R4 bundle** before synchronization.

10. **We press "Sync & finish visit".**

    * *Right now:* Four progress lines animate with a fake delay, and then we see a random Visit ID.
    * *Future:* We will call the real **ABDM M2 and M3 APIs**, which will permanently link this visit to the patient's central ABHA record.

The **"System activity" log panel** on the right records one line for every click, along with a CLIENT / AUTH / AI / OCR / SYNC tag. This helps us understand which production component from our architecture will be called behind each action. The **"What's simulated?"** button in the top bar shows the complete list in one place.

---

## Before We Start: Access We Cannot Skip

These are not libraries we can simply `npm install` — they are external approvals. We should apply for them on day one because they can take the longest.

| # | What                                      | Where                                                                                                                                      | Why we need it                                                                                                                                                                                                                                                                              |
| - | ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1 | **ABDM Sandbox account**                  | `sandbox.abdm.gov.in` — fill the sandbox registration form                                                                                 | Gives us sandbox credentials for ABHA creation/verification (Milestone 1) and later M2/M3 data-push APIs. Approval typically takes a few days.                                                                                                                                              |
| 2 | **UIDAI Aadhaar e-KYC / OTP access**      | Only available to a licensed **AUA/KUA** (Authentication User Agency / KYC User Agency) under UIDAI                                        | We almost certainly will **not** get direct production Aadhaar access as an individual or hackathon team. For a prototype/pilot, we should create ABHA through the **ABDM Sandbox's own enrolment flow**, which handles the Aadhaar OTP step on our behalf — we do not call UIDAI directly. |
| 3 | **Bhashini / ULCA account**               | `bhashini.gov.in` → sign up → generate `userID` + `ulcaApiKey` (+ an inference API key) from "My Profile"                                  | We need this for real speech-to-text and text-to-speech in Indian languages, replacing the prototype's canned transcript.                                                                                                                                                                   |
| 4 | **NAMASTE Portal terminology access**     | `namaste.ayush.gov.in`                                                                                                                     | This is where the standardized AYUSH morbidity codes (used to tag Ayurvedic diagnoses) are published. We need the code list/API access from here to code AYUSH findings consistently.                                                                                                       |
| 5 | **SNOMED CT (India) access**              | Distributed via the **National Resource Centre for EHR Standards (NRCeS), C-DAC Pune**, which also maintains the SNOMED CT AYUSH extension | We need this to code allopathic (and cross-walked AYUSH) findings to a standard terminology.                                                                                                                                                                                                |
| 6 | **AWS account** (only if we use Textract) | `aws.amazon.com`                                                                                                                           | We need this if we choose AWS Textract for OCR. We can skip this entirely if we use Tesseract instead (see Step 4).                                                                                                                                                                         |

Everything else below (frameworks, databases, servers) is under our control, so we can start working on it immediately in parallel with these applications.

---

## Step 1 — Scaffold the Project

We should keep the four architecture layers as four separate, independently runnable pieces from day one. This will save us from creating a tangled monolith later.

```text
swasthsetu/
  kiosk-app/       -> Layer 1: the touchscreen frontend (this is what swasthsetu-prototype.html stands in for)
  auth-service/    -> Layer 2: identity & auth
  clinical-engine/ -> Layer 3: AI triage / AYUSH assessment logic
  sync-service/    -> Layer 4: OCR + FHIR + ABDM sync
```

Each service will be its own small backend that our kiosk app talks to over plain HTTP/JSON. This mirrors our four-layer diagram directly, so anyone joining the project can easily see which folder implements which box in the architecture.

---

## Step 2 — Build the Kiosk Frontend (Layer 1)

This is the part we already have a working reference for: `swasthsetu-prototype.html` shows exactly what every screen should look like and in what order.

1. We initialize a React project (e.g. with Vite) inside `kiosk-app/`.
2. We add Tailwind CSS for styling.
3. We rebuild each screen from the prototype as a React component: Welcome → Patient Type → Identity → Mode Select → Complaint → Assessment (AYUSH or General) → Follow-up (returning patients only) → Reports → Summary → Sync → Done.
4. We replace the prototype's mocked logic with real HTTP calls to the three backend services we build in Steps 3–5.
5. Once the web app runs correctly in a normal browser, we wrap it with **Electron.js** to turn it into a locked-down kiosk application (full-screen, no browser chrome, no way to navigate away).
6. We wire in the **Bhashini ULCA API** for the mic button (speech-to-text) and for reading prompts/results aloud (text-to-speech), using the `userID`/`ulcaApiKey` from our Bhashini sign-up.

At the end of this step, we will have a real, running kiosk UI — it just won't verify identity, assess patients, or sync anywhere yet. Those are covered in Steps 3–5.

---

## Step 3 — Build the Identity & Auth Service (Layer 2)

This service will do three jobs: create an ABHA ID for new patients, verify an existing one for returning patients, and issue a session token that the rest of the kiosk visit will use.

1. We stand up a small backend (Node.js/Express or Python/FastAPI — either is fine; we can pick whichever our team knows).
2. We integrate the **ABDM Sandbox's ABHA enrolment APIs** (Milestone 1) using the sandbox credentials from our registration. This flow handles Aadhaar-based ABHA creation for new patients without us touching UIDAI directly.
3. For returning patients, we integrate the **ABHA verification / profile-fetch** endpoints from the same sandbox to pull their existing SwasthSetu profile.
4. We issue a short-lived session using **OAuth 2.0 + JWT** once identity is confirmed. Every later call from the kiosk (to the clinical engine and sync service) should carry this token.
5. We test this service entirely against the ABDM **sandbox** environment first — we do not attempt production ABDM access until the sandbox flow works end to end and we have gone through ABDM's exit/production process.

---

## Step 4 — Build the Clinical Engine (Layer 3)

This service takes the patient's complaint and assessment answers and turns them into a coded clinical finding — differently for AYUSH and allopathic mode.

1. We stand up a Python service (FastAPI is a natural fit here, since most open medical NLP models are Python-first).
2. For symptom/complaint understanding, we host an open medical language model such as **BioMistral** (a Mistral-based biomedical LLM available on Hugging Face) or a comparable **Med-Llama**-family model, and expose it behind a simple `/extract-symptoms` endpoint.
3. For AYUSH mode: we implement the **Dashavidha Pariksha** as an explicit rules layer — a set of questions/inputs (Prakriti, Vikriti, Sara, Samhanan, Satmya, Satva, Ahara Shakti, Vyayama Shakti, Vaya, Pramana) that combine into a structured result, the same way the prototype's sliders and chips do, just with real clinical logic behind them instead of a placeholder calculation. We have an AYUSH clinician validate this rules layer — this is not something we should design purely from documentation.
4. We code the AYUSH output against the **NAMASTE** terminology (Step 0, item 4) and the allopathic output against **SNOMED CT** (item 5), so both modes produce standard, machine-readable diagnosis codes rather than free text.
5. We return a structured JSON result (codes + a human-readable summary) to the kiosk frontend and onward to the sync service.

---

## Step 5 — Build the OCR & Sync Service (Layer 4)

This service digitizes paper reports and writes the finished visit to the patient's central ABHA record.

1. We stand up a backend for this service, with a local database — **MongoDB or PostgreSQL** — as the kiosk-side cache. This cache exists so a visit isn't lost if the internet connection to ABDM drops mid-consultation.
2. For OCR, we choose one:

   * **Tesseract** (free, open-source, runs entirely on our own server — good starting point since it needs no external account), or
   * **AWS Textract** (a paid AWS service, generally more accurate on messy scans, needs the AWS account from Step 0).
3. We assemble the finished visit record as an **HL7 FHIR R4** bundle. There are open FHIR libraries in most languages (e.g. `fhir.resources` in Python, HAPI FHIR in Java) — we should use one rather than hand-building the JSON structure, since FHIR resources have many required fields.
4. We push that bundle through the **ABDM M2 and M3 APIs** (from our sandbox registration) to link it to the patient's central ABHA record.
5. Once the push succeeds, we clear the local cache entry for that visit.

---

## Step 6 — Wire It All Together

1. We point the kiosk frontend's API calls at our four real services instead of the mocked logic in the prototype.
2. We walk through the exact same journey the prototype demonstrates — patient type → identity → mode → complaint → assessment → follow-up → reports → summary → sync — but now each step is a real network call.
3. We use `swasthsetu-prototype.html` side-by-side as our acceptance test: for every screen, the real app should ask the same questions, in the same order, and end with the same outcome (a synced ABHA record) as the prototype simulates.
4. We test the **offline/poor-connectivity case** deliberately: we disconnect the network mid-visit and confirm the kiosk-side cache (Step 5) holds the visit and retries the sync once connectivity returns, instead of losing the data.

---

## Step 7 — Before Any Real Patient Uses It

* We get sign-off that the deployment is reviewed against the **Digital Personal Data Protection Act, 2023**, since this handles real patients' health data.
* We confirm that we have captured the **ABDM consent-manager** artifacts required before fetching or pushing a patient's health data — this is a formal part of the ABDM flow, not optional.
* We complete the ABDM **sandbox exit process** (their documented path from sandbox to production) before going live in a real clinic — we cannot point a live kiosk at the sandbox environment.
* We have a clinician review the Dashavidha Pariksha rules and the AI-suggested triage output before it is used to inform actual care decisions — none of the AI/rules components here should make an unreviewed clinical decision.

---

## Exact Tech Stack

This is the stack as specified in our architecture, with the concrete tools/services we would use to implement each box.

| Layer                               | Component                    | Concrete tech                                                                                                            |
| ----------------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **1 · Client & Hardware**           | Frontend framework           | React.js                                                                                                                 |
|                                     | Styling                      | Tailwind CSS                                                                                                             |
|                                     | Kiosk wrapper                | Electron.js                                                                                                              |
|                                     | Voice input                  | Bhashini ASR (via ULCA API) or OpenAI Whisper                                                                            |
|                                     | Voice output                 | Bhashini TTS (via ULCA API) or ElevenLabs                                                                                |
| **2 · Identity & Auth**             | ABHA creation/verification   | ABDM Sandbox APIs (Milestone 1), at `sandbox.abdm.gov.in`                                                                |
|                                     | Aadhaar-based enrolment      | Handled through the ABDM Sandbox enrolment flow (not direct UIDAI access, unless our organization is a licensed AUA/KUA) |
|                                     | Returning-patient data fetch | ABHA Sandbox Gateway                                                                                                     |
|                                     | Session security             | OAuth 2.0 + JWT                                                                                                          |
| **3 · AI Triage & Clinical Engine** | Symptom/complaint extraction | BioMistral or a Med-Llama–family open medical LLM, served from a Python backend                                          |
|                                     | AYUSH coding                 | NAMASTE Portal terminology (`namaste.ayush.gov.in`)                                                                      |
|                                     | AYUSH assessment logic       | Custom rules engine implementing the Dashavidha Pariksha, validated by a clinician                                       |
|                                     | Allopathic coding            | SNOMED CT, distributed via NRCeS/C-DAC Pune                                                                              |
| **4 · OCR & Central Sync**          | Report digitization          | Tesseract (open-source) or AWS Textract (paid)                                                                           |
|                                     | Clinical data format         | HL7 FHIR R4 (use an existing FHIR library, don't hand-build the JSON)                                                    |
|                                     | Kiosk-local cache            | MongoDB or PostgreSQL                                                                                                    |
|                                     | Central record push          | ABDM M2 and M3 APIs                                                                                                      |
| **Cutting across all layers**       | Backend language/framework   | Node.js/Express or Python/FastAPI for the three backend services                                                         |
|                                     | Version control & CI         | Git, with a repo laid out as in Step 1                                                                                   |
