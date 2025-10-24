Technical Specification: Agentic Docs Builder - Flora Edition
1. Introduction
The Agentic Docs Builder is a web-based, single-page application (SPA) designed to facilitate the automated generation of documents from structured data. It provides a user-friendly, five-step workflow that guides users through uploading data, defining a template, generating initial documents, configuring a chain of AI agents, and finally, executing an AI-powered processing pipeline on the content. The system leverages the Google Gemini API to provide intelligent text transformation and analysis capabilities.
2. System Architecture
2.1. Frontend Framework
Core: The application is built as a Single Page Application using React 18 and TypeScript.
Rendering: It uses react-dom/client for modern concurrent rendering.
State Management: State is managed locally within the main App component using React Hooks (useState, useCallback, useMemo). There is no external state management library like Redux or Zustand.
2.2. Styling & UI
Framework: TailwindCSS is used for all styling, providing a utility-first approach for a responsive and consistent design.
Theme: A custom color palette (Primary: #E6E6FA, Accent: #9370DB, etc.) is configured directly within the index.html file.
Component Structure: The UI is composed of reusable, stateless functional components (Card, TabButton, FileInput) and SVG icons defined as React components for modularity and clarity.
2.3. Dependencies
AI Backend: @google/genai for communication with the Google Gemini API.
File Parsing:
xlsx: For parsing .xlsx and .ods spreadsheet files.
papaparse: For parsing .csv files.
mammoth.js: For extracting raw text from .docx files.
Core Runtime: React and ReactDOM are loaded via CDN.
3. Core Features & Components
The application's workflow is divided into five distinct tabs.
3.1. Tab 1: Data Ingestion
Purpose: To upload and parse a structured dataset.
UI Components:
FileInput: A drag-and-drop enabled file upload component.
Data Table: A preview table showing the first 10 records of the loaded dataset, with headers derived from the data schema.
Logic (handleDatasetFile):
Receives a File object from the FileInput component.
Invokes fileParserService.parseDatasetFile to process the file based on its extension.
Supported formats: CSV, JSON, XLSX, ODS, TXT.
On success, it updates the dataset state with an array of DatasetRecord objects and the schema state with an array of column headers.
Handles and displays parsing errors.
3.2. Tab 2: Template Definition
Purpose: To define the structure of the output documents using a template.
UI Components:
FileInput for uploading template files (.txt, .md, .docx).
A textarea for pasting or directly editing the template text.
A "Live Preview" panel that shows the template rendered with data from the first record of the dataset.
Logic:
Templating Syntax: Uses {{column_name}} placeholders for data injection.
Live Preview: A replace function runs on the template string, substituting placeholders with corresponding values from dataset[0].
Document Generation (handleGenerateDocs): When the "Generate Documents" button is clicked, it iterates through every record in the dataset, applies the template, and populates the generatedDocs state.
3.3. Tab 3: Generated Documents
Purpose: To review and edit the documents created in the Template step.
UI Components:
A scrollable list of collapsible sections (<details>), one for each generated document.
Each section contains the document's filename and a textarea with its content.
Logic: The content within each textarea is bound to the corresponding GeneratedDoc object in the generatedDocs state, allowing for direct, in-place editing.
3.4. Tab 4: Agent Configuration
Purpose: To define and configure a sequence of AI agents for a processing pipeline.
UI Components:
A list of collapsible sections, one for each agent.
Form fields within each section allow modification of the Agent properties (name, model, temperature, prompts, etc.).
Logic:
The agents state is initialized with three DEFAULT_AGENTS_CONFIG: a Summarizer, a Style Rewriter, and a JSON Converter.
The handleAgentChange function updates the agents state array when a user modifies any agent parameter. This allows for dynamic configuration of the pipeline before execution.
3.5. Tab 5: Pipeline Execution
Purpose: To run a selected input through the configured agent pipeline and view the results.
UI Components:
An input textarea for the initial text.
Helper buttons to quickly populate the input from the first few generated documents.
An "Execute Pipeline" button, which is disabled during execution.
A results panel that displays the output of each agent step-by-step.
A "Follow-up Questions" card that appears upon successful completion.
Logic (startPipeline):
Sets isPipelineRunning to true and clears previous results.
Iterates through the agents array sequentially.
For each agent, it calls geminiService.runAgent with the output of the previous step as the input for the current one.
Updates the pipelineHistory state after each step completes, causing the UI to re-render and show progress.
If any agent returns an error, the pipeline stops.
After the final agent successfully runs, it calls geminiService.generateFollowUpQuestions with the final output to provide contextual next steps.
Sets isPipelineRunning to false upon completion or error.
4. Services
4.1. geminiService.ts
Purpose: Encapsulates all interactions with the Google Gemini API.
Initialization: Implements a singleton pattern (getAi) to initialize the GoogleGenAI client once using process.env.API_KEY.
runAgent(agent, input):
Takes an Agent configuration object and an input string.
Uses renderTemplate to inject the input into the agent's user_prompt.
Calls ai.models.generateContent with the agent's model, prompts, and parameters (temperature, max_tokens, topP).
Returns the .text property from the API response or a formatted error string.
generateFollowUpQuestions(context):
Takes a context string (the final pipeline output).
Calls the gemini-2.5-flash model with a hardcoded prompt to generate three questions.
Returns the text response.
4.2. fileParserService.ts
Purpose: Abstracts the logic for reading and parsing different file types.
parseDatasetFile(file):
Uses a switch statement on the file extension.
Delegates parsing to the appropriate library (PapaParse for CSV, xlsx for spreadsheets, JSON.parse for JSON).
Returns a Promise<DatasetRecord[]>.
parseTemplateFile(file):
Handles .txt, .md, and .docx files.
Delegates .docx parsing to mammoth.js.
Returns a Promise<string> containing the file's text content.
5. Data Models (types.ts)
DatasetRecord: Record<string, string | number | boolean> - Represents a single row in the input dataset.
Agent: An interface defining the configuration for an AI agent, including its name, prompts, model, and generation parameters.
GeneratedDoc: An interface for a document produced from the template, containing its content and filename.
PipelineStep: Represents a single step in the execution history, storing the agent's name, input, output, and any potential error.
Tab: An enum (Data, Template, Generate, Agents, Run) for managing the active UI tab.
