# RHOAM ARCHITECTURE - 29 CRITICAL QUESTIONS

## I. RHOAM Core Logic & Data (The Engine)
1. Encryption Key Management: How is the master encryption key derived and secured?
2. API Orchestration & Rate Limiting: What are the exact minimum time intervals that the rate limiter must enforce between API calls?
3. Failure Handling: How does RHOAM Core handle an individual bot API call failure?
4. Response Order: How does RHOAM Core guarantee that the array of 9 responses is returned in the correct order?
5. Data Persistence (Settings & History): What is the exact path and naming convention for the encrypted settings file?
6. Core Class Structure: What are the minimal required classes within the Node.js RHOAM Core?
7. Configuration Loading: How does RHOAM Core load its base configuration before the encrypted user settings file is accessed?
8. API Key Validation: What is the simplest, non-intrusive method for RHOAM Core to validate that a newly entered API key is syntactically correct?
9. Error Object Standardization: What is the uniform JSON error object structure that RHOAM Core will return to the UI for any failure?
10. Streaming vs. Batch: Should RHOAM Core implement response streaming or stick strictly to synchronous batch completion?
11. Dependency Minimization: What is the absolute minimum external Node.js dependency set required?
12. Rate Limiter Implementation: Will the Rate Limiter use a global lock or be instantiated per-API-key?

## II. RHOAM UI/UX (The Interface & Integration)
13. Integration Protocol: What is the precise JSON structure for the two main message types?
14. Error Display: How are core errors communicated back to the UI, and what is the visual display mechanism?
15. Visual Specification (The 3x3 Grid): What input type will be used for the 9 API keys, and will there be a visual indicator of successful connection?
16. Color Assignment: How are the 9 unique bot colors defined?
17. VS Code Wrapper (The Thin Layer): What is the final build output directory for the compiled Svelte RHOAM UI?
18. Compilation Step: Does the VS Code extension's build process automatically trigger the RHOAM UI's build process?
19. Svelte Store: How will global state be managed within the Svelte RHOAM UI?
20. Message Passing: How will the RHOAM UI ensure that a button click immediately triggers the correct PostMessage event?
21. Chat Rendering Efficiency: What is the optimal Svelte component structure for rendering the chat history efficiently?
22. CSS Isolation: How will the RHOAM UI ensure its custom styling is perfectly isolated?
23. Webview-to-Core Initialization: What is the precise handshake sequence that occurs when the Webview loads?
24. Settings Persistence Trigger: What is the single, non-redundant user action that triggers the UI to send the encrypted settings back to RHOAM Core?

## III. VC-Grade Standards & Deployment (The Process)
25. Testing Strategy: Which testing framework will be used for both the RHOAM Core and the RHOAM UI?
26. CI/CD Pipeline (Automated Deployment): What is the exact sequence of steps for the GitHub Actions workflow to automatically deploy RHOAM?
27. Automated Testing Triggers: Which GitHub Actions events will be configured to automatically trigger the full Unit Test suite?
28. Build Artifact Validation: How will the CI/CD pipeline confirm that the final VSIX package contains all required assets?
29. FOSS Governance & Licensing: Will a simple Contributor License Agreement (CLA) be required, or will a standard FOSS license be sufficient?
