Architecture

    When a text is submitted via POST /submit, the application pipes the raw text into the Groq API handler and the local stylometric math function. The resulting values are combined into a final confidence score, mapped to a UX text label, written as a structured record to our log ledger, and returned to the caller. 

    Submission Flow
    ---
    config:
    theme: redux
    look: classic
    fontFamily: '''Open Sans Variable'', sans-serif'
    themeVariables:
        fontFamily: '''Open Sans Variable'', sans-serif'
    layout: dagre
    ---
    flowchart LR
        A["POST /submit"] --> B["Flask Ingestion"]
        B --> C1["Groq LLM Engine"] & C2["Stylometrics Engine"]
        C1 -- LLM Score --> D["Scoring Engine"]
        C2 -- Variance Score --> D
        D -- Combined Score --> E["Label Matcher"]
        E -- Text Label --> F["Save JSON Log"]
        F --> G["Client Response"]


    If a creator contests the decision, POST /appeal finds that ledger entry, flips its state to "under_review", and attaches the creator's reasoning.

    Appeal Flow

    ---
    config:
    theme: redux
    ---
    graph LR
        A[POST /appeal] --> B[Flask Appeal Endpoint]
        B --> C{Find content_id?}
        C -->|Yes| D[Update Status to under_review]
        C -->|No| E[Return 404 Error]
        D --> F[Append Reason to Log]
        F --> G[Return 200 Confirmation]


Detection Signals & Combination Logic

    Signal 1: Groq LLM Semantic Analysis
        - Semantic predictability, tonal uniformity, and reliance on common AI transition words.
        - Output format: A float value between 0.0 (pure human) and 1.0 (pure AI).
    
    Signal 2: Stylometric Sentence Length Variance
        - The statistical variance of word counts per sentence. AI text usually hits a uniform rhythm (low variance). Human text naturally swings between short bursts and long clauses (high variance).
        - Output format: A float value between 0.0 and 1.0. We calculate the text's actual variance and map it: very low variance yields a score near 1.0 (likely AI), while high variance yields a score near 0.0 (likely human since it is more chaotic).

    Combination Formula Final Score = (Signal 1 * 0.55) + (Signal 2 * 0.45)
    Changed Combination Formula Final Score = (Signal 1 * 0.6) + (Signal 2 * 0.4) - I changed it because signal 2 had too much influence on the final score considering its relevance.

Uncertainty Representation & Thresholds

    A final confidence score of 0.6 means the system detected conflicting indicators; for instance, a text containing highly varied sentence lengths (human trait) that simultaneously uses repetitive, cliché AI phrases maybe because that is the persons writing style. Or it could be a highly stylized AI response tailored to sound human.

    To prevent false accusations against authentic creators, the categorization deliberately avoids a strict 0.5 binary flip:

        Score 0.00 to 0.39: Likely Human

        Score 0.40 to 0.70: Uncertain / Mixed Signals

        Score 0.71 to 1.00: Likely AI-Generated

Transparency Label Design

    These exact text labels will be returned by the API and displayed to readers on the platform:

        High-Confidence Human Result (0.00 - 0.39):

        - "Human Generated: This content matches natural human writing patterns with high structural diversity and authenticity."

        Uncertain Result (0.40 - 0.70):

        - "Unverified Attribution: This text exhibits mixed stylistic patterns, combining automated uniformity with creative variation."

        High-Confidence AI Result (0.71 - 1.00):

        - "AI-Generated: This content displays high predictability and structural regularity characteristic of automated text models."

Appeals Workflow

    Who can appeal: Any platform creator whose content receives an "AI-Generated" or "Unverified Attribution" label.

    Information provided: The creator must pass the content_id and a short string explaining their case.

    System Actions: 
    1. Receives the request at POST /appeal.
    2. Searches the audit log for the corresponding content_id.
    3. Changes the log status field from "classified" to "under_review".
    4. Appends the creator_reasoning and an appealed_at timestamp directly to that record.

    Reviewer View: A human administrator pulling the queue sees an array of items where status == "under_review", displaying the original text snippet, the conflicting signal scores, and the creator's written defense.

Anticipated Edge Cases
    
    Polished Academic or Technical Essays: Human-written research papers often follow strict format. This creates flat sentence variations and predictable vocabularies, which may trick both signals into throwing a false AI flag.

    Mixed AI/Human Text: A creator prompts an LLM to draft an article, then manually rewrites parts of it, swaps out common vocabulary, and injects personal anecdotes to bypass the system. This structural blending directly challenges threshold boundaries, likely forcing the system into the "Uncertain" range and increasing the volume of manual appeals confusing the system and reviewers.

    Short Poetry: Text submissions under 50 words don't provide enough data points.

AI Tool Plan

    Milestone 3: Submission Endpoint & First Signal
    Sections to provide to AI: ## Architecture (Narrative + Diagram) + Detection Signals (Signal 1 details).

    Prompt Request: 
        CONTEXT:
        I am building "Provenance Guard," a backend system using Python and Flask to analyze whether submitted text is human-written or AI-generated. 

        Here is my core submission architecture narrative:
        When text is submitted via POST /submit, the application routes the raw text to a Groq LLM API handler (Signal 1). The system assigns a unique UUID content_id, logs the initial results into a structured JSON file/list audit log, and returns the response.

        I am using the following libraries: flask, groq (v0.15.0), python-dotenv.

        INPUT:
        A JSON payload to a POST /submit endpoint containing:
        - "text": string (the content to analyze)
        - "creator_id": string (the unique identifier of the user)

        OUTPUT REQUIRED:
        Please provide the complete, clean Python code for app.py that includes:
        1. Flask app initialization that loads environment variables via python-dotenv for GROQ_API_KEY.
        2. A standalone function `analyze_semantic_signal(text)` that sends the text to the Groq API using model "llama-3.3-70b-versatile". Use a system prompt forcing the model to evaluate the text and return ONLY a valid JSON object matching this structure: {"ai_probability": 0.85}. Parse this JSON response and return the float.
        3. A POST /submit endpoint that generates a string UUID content_id, calls `analyze_semantic_signal(text)`, and returns a 200 OK JSON response containing:
        - "content_id": string (the generated UUID)
        - "attribution": string (if score > 0.5 "likely_ai" else "likely_human")
        - "confidence": float (the raw score from Groq)
        - "label": "PLACEHOLDER_LABEL"
        4. A simple in-memory or text-file based JSON audit log tracker. Every call to POST /submit must append a record containing: content_id, creator_id, timestamp, attribution, confidence, llm_score (the raw Groq score), and a status of "classified".
        5. A GET /log endpoint that reads and returns all logged records inside a JSON object: {"entries": [...]}.

        Ensure the code includes robust error handling for the API call and JSON parsing. Add comments explaining important explaning important code and logic sections for my learning. Do not use markdown UI components, just raw backend logic.

    Verification: Run the app locally, send a sample text snippet using curl, and print the raw float output from the Groq helper function to confirm it connects and parses cleanly.

    Milestone 4: Second Signal & Confidence Scoring
    Sections to provide to AI: Detection Signals (Signal 2 details) + Uncertainty Representation & Thresholds.

    Prompt Request: 
        CONTEXT:
        I am upgrading my existing Provenance Guard Flask application to support a multi-signal pipeline and specific scoring thresholds. I will paste my current app.py code below this prompt.

        I need to integrate a pure-Python second signal (Stylometrics) and calculate a calibrated weighted score based on these exact rules from my spec:
        - Signal 2 (Stylometrics) measures sentence length variance. It splits the text into sentences, counts words per sentence, and computes variance. Low variance indicates uniform AI writing; high variance indicates natural human writing. This variance must be normalized to a 0.0 (highly varied/human) to 1.0 (highly uniform/AI) scale.
        - The combination formula is: Final Score = (Groq Score * 0.55) + (Stylometric Score * 0.45)
        - Calibration thresholds:
        - 0.00 to 0.39: "likely_human"
        - 0.40 to 0.70: "uncertain"
        - 0.71 to 1.00: "likely_ai"

        INPUT:
        [PASTE YOUR WORKING APP.PY CODE FROM MILESTONE 3 HERE]

        OUTPUT REQUIRED:
        Please update the app.py code to include:
        1. A standalone function `calculate_stylometric_variance(text)` using pure Python (no external NLP libraries like nltk/spaCy). It should compute sentence length variance and output a normalized float between 0.0 and 1.0 (where 1.0 is maximum uniformity/lowest variance). Handle short inputs gracefully.
        2. An updated POST /submit flow that executes both `analyze_semantic_signal(text)` and `calculate_stylometric_variance(text)`.
        3. An explicit scoring block applying the weighted formula to compute the final confidence score, mapping it to the correct attribution flag ("likely_human", "uncertain", "likely_ai") using the thresholds defined in the context.
        4. An updated audit log entry schema that records both individual scores alongside the combined metrics:
        {
            "content_id": "...",
            "creator_id": "...",
            "timestamp": "...",
            "attribution": "likely_ai" | "uncertain" | "likely_human",
            "confidence": final_score,
            "llm_score": groq_score,
            "stylometric_score": variance_score,
            "status": "classified"
        }
        Ensure all values inside the JSON responses match these newly updated variables.

    Verification: Feed the scoring script 4 manual test inputs (1 obvious AI block, 1 messy human text, 1 formal essay, 1 short poem) and verify that the combined score falls inside the expected threshold brackets.

    Milestone 5: Production Layer
    Sections to provide to AI: Transparency Label Design + Appeals Workflow + ## Architecture (Appeal Flow Diagram).

    Prompt Request: 
        CONTEXT:
        I am adding the final production layer to my Provenance Guard Flask app. I will paste my current code below. 

        I need to implement user-facing transparency labels, an asynchronous-style appeal system, and endpoint protection via Flask-Limiter. Here are my spec constraints:
        - Label Text Mapping based on final score:
        High-Confidence Human Result (0.00 - 0.39):
        - "Human Generated: This content matches natural human writing patterns with high structural diversity and authenticity."
        Uncertain Result (0.40 - 0.70):
        - "Unverified Attribution: This text exhibits mixed stylistic patterns, combining automated uniformity with creative variation."
        High-Confidence AI Result (0.71 - 1.00):
        - "AI-Generated: This content displays high predictability and structural regularity characteristic of automated text models."

        - Appeal Workflow:
        - Creators submit contested content to a POST /appeal endpoint. 
        - The server searches the audit log for the content_id, updates its status from "classified" to "under_review", and appends the creator's explanation.
        - Rate Limiting:
        - Use flask-limiter (v3.5.0+) with local in-memory storage: Limiter(get_remote_address, app=app, storage_uri="memory://")
        - Set a threshold of 10 requests per minute and 100 requests per day specifically on the POST /submit endpoint.

        INPUT:
        [PASTE YOUR WORKING APP.PY CODE FROM MILESTONE 4 HERE]

        OUTPUT REQUIRED:
        Please update the code to implement:
        1. A helper function that returns the exact verbatim string label based on the final confidence score thresholds. Replace the "PLACEHOLDER_LABEL" string inside POST /submit with this runtime output.
        2. A new POST /appeal endpoint that accepts a JSON object containing:
        - "content_id": string
        - "creator_reasoning": string
        3. Within the POST /appeal route, look up the target entry in your audit log ledger. Update its "status" field to "under_review" and inject two new fields into that logged entry: "creator_reasoning" (the passed string) and "appealed_at" (current ISO timestamp). Return a 200 OK success JSON response if found, or a 404 error if the content_id doesn't exist.
        4. Integrate Flask-Limiter on the POST /submit route matching the parameters provided in the context, ensuring it does not crash or throw startup configuration warnings.

    Verification: Send a payload to /submit, grab the returned content_id, pass it to /appeal with a test reason string, and then call GET /log to verify that the record successfully updated its status and stored your reason.