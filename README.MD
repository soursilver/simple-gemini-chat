# Simple Gemini API Chat (Svelte + DaisyUI)

## Description

This is a simple, web-based chat application built with Svelte, Vite, and DaisyUI. It serves as a basic starter project to demonstrate how to connect to and interact with Google's Gemini AI models using the official `@google/genai` JavaScript SDK, specifically focusing on **streaming responses**.

The primary goal of this app is to provide a clean, functional example for developers looking to quickly get up and running with the Gemini API in a modern web framework. It showcases handling API keys, selecting models, managing chat history locally, and rendering streamed AI responses in real-time.

## Features

*   **Real-time Chat Interface:** Clean and responsive chat UI built with DaisyUI components.
*   **Gemini AI Integration:** Connects to the Google Generative AI API.
*   **Streaming Responses:** Correctly handles and displays streamed responses from the Gemini model using `ai.models.generateContentStream`.
*   **Model Selection:** Allows users to choose from a predefined list of Gemini models (e.g., `gemini-1.5-flash-latest`, `gemini-1.5-pro-latest`).
*   **Custom Model Addition:** Users can add custom model names via the UI, which are persisted in the browser's `localStorage`.
*   **Chat History:**
    *   Sidebar displays a list of previous chat sessions.
    *   Ability to start new chats.
    *   Loads previously selected chats.
    *   Chat history is persisted in the browser's `localStorage`.
*   **Basic State Management:** Uses Svelte's built-in reactivity and component structure for state.
*   **Environment Variable Handling:** Securely loads the Gemini API key from a `.env` file.

## Getting Started

### Prerequisites

*   Node.js (LTS version recommended)
*   npm, yarn, or pnpm package manager
*   A Google Gemini API Key

### Setup

1.  **Clone the repository:**
    ```bash
    git clone <your-repository-url>
    cd <repository-directory>
    ```

2.  **Install dependencies:**
    ```bash
    npm install
    # or
    yarn install
    # or
    pnpm install
    ```

3.  **Set up Environment Variables:**
    *   Create a file named `.env` in the root directory of the project.
    *   Add your Google Gemini API key to this file. You can obtain a key from [Google AI Studio](https://aistudio.google.com/app/apikey).
    *   **Important:** Vite requires environment variables exposed to the client-side code to be prefixed with `VITE_`.

    **.env file contents:**
    ```env
    VITE_GEMINI_API_KEY=YOUR_API_KEY_HERE
    ```
    Replace `YOUR_API_KEY_HERE` with your actual API key.

### Running the App

1.  **Development Mode:**
    *   Starts a local development server with Hot Module Replacement (HMR).
    *   Access the app at `http://localhost:5173` (or the port specified in your terminal).
    ```bash
    npm run dev
    ```

2.  **Building for Production:**
    *   Creates an optimized build of the application in the `dist` folder.
    ```bash
    npm run build
    ```

3.  **Previewing the Production Build:**
    *   Starts a simple local server to serve the contents of the `dist` folder.
    ```bash
    npm run preview
    ```

## Technology Stack

*   **Svelte:** UI Framework
*   **Vite:** Build Tool & Development Server
*   **Tailwind CSS:** Utility-First CSS Framework
*   **DaisyUI:** Tailwind CSS Component Library
*   **@google/genai:** Google Generative AI SDK for JavaScript
*   **Browser `localStorage`:** For persisting chat history and custom models locally.

## Help From Gemini

The app itself was build with help from Gemini, which also wrote this readme file.