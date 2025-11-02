<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Original Character Maker</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@100..900&family=Playfair+Display:ital,wght@0,400..900;1,400..900&display=swap');
        
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f7f7f7;
            display: flex;
            justify-content: center;
            padding: 1rem;
        }

        .container {
            max-width: 768px;
            width: 100%;
            background: #ffffff;
            box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1);
            border-radius: 16px;
            padding: 2rem;
        }

        .title-font {
            font-family: 'Playfair Display', serif;
        }

        /* Custom scrollbar for output */
        #output-area::-webkit-scrollbar {
            width: 8px;
            height: 8px;
        }
        #output-area::-webkit-scrollbar-thumb {
            background-color: #6d28d9; /* Violet-700 */
            border-radius: 10px;
        }
        #output-area::-webkit-scrollbar-track {
            background: #e5e7eb; /* Gray-200 */
            border-radius: 10px;
        }

        /* Loading Spinner */
        .spinner {
            border: 4px solid rgba(255, 255, 255, 0.3);
            border-radius: 50%;
            border-top: 4px solid #fff;
            width: 24px;
            height: 24px;
            animation: spin 1s linear infinite;
            margin-right: 0.5rem;
        }

        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
    </style>
</head>
<body class="bg-gray-50 min-h-screen">

    <div class="container my-6">
        <header class="text-center mb-8">
            <h1 class="text-4xl font-extrabold title-font text-violet-800">
                The OC Forge
            </h1>
            <p class="text-gray-600 mt-2">
                Craft compelling character profiles with the help of Gemini.
            </p>
        </header>

        <!-- Input Form -->
        <div class="space-y-4 mb-8 p-4 bg-violet-50 rounded-lg border border-violet-200">
            <div>
                <label for="charName" class="block text-sm font-medium text-gray-700">Character Name (Optional)</label>
                <input type="text" id="charName" placeholder="E.g., Kaelen, Subject 7"
                       class="mt-1 block w-full rounded-md border-gray-300 shadow-sm p-3 border focus:ring-violet-500 focus:border-violet-500 transition duration-150">
            </div>
            <div>
                <label for="charGenre" class="block text-sm font-medium text-gray-700">Genre (Required)</label>
                <input type="text" id="charGenre" placeholder="E.g., Cyberpunk, High Fantasy, Historical Romance"
                       class="mt-1 block w-full rounded-md border-gray-300 shadow-sm p-3 border focus:ring-violet-500 focus:border-violet-500 transition duration-150">
            </div>
            <div>
                <label for="charConcept" class="block text-sm font-medium text-gray-700">Specific Character Details (Optional: Appearance, Powers, Conflict, etc.)</label>
                <textarea id="charConcept" rows="3" placeholder="E.g., Wears a tattered red cloak, can manipulate shadows, seeks revenge for a fallen mentor, has a prosthetic arm. (Leave blank for a random concept)"
                          class="mt-1 block w-full rounded-md border-gray-300 shadow-sm p-3 border resize-none focus:ring-violet-500 focus:border-violet-500 transition duration-150"></textarea>
            </div>

            <!-- Event listener attached programmatically in script -->
            <button id="generateButton"
                    class="w-full flex items-center justify-center px-4 py-3 border border-transparent text-base font-medium rounded-lg shadow-lg text-white bg-violet-600 hover:bg-violet-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-violet-500 transition duration-300 ease-in-out">
                Generate Character Profile
            </button>
        </div>

        <!-- Output and Loading Area -->
        <div class="mt-8">
            <h2 class="text-2xl font-bold title-font text-violet-800 mb-4">Generated Profile</h2>
            <div id="loading-indicator" class="hidden flex items-center justify-center p-6 bg-yellow-50 text-yellow-800 rounded-lg shadow-inner">
                <div class="spinner border-violet-300 border-t-violet-600"></div>
                <p class="ml-3 font-semibold">Forging character details... please wait.</p>
            </div>

            <div id="output-area" class="min-h-48 p-6 bg-white rounded-lg border-2 border-dashed border-gray-300 text-gray-700 leading-relaxed overflow-y-auto max-h-96">
                <p class="text-center text-gray-400 italic">
                    Your original character profile will appear here after generation.
                </p>
            </div>
            
            <!-- Citation Sources -->
            <div id="source-list" class="mt-4 p-3 bg-gray-100 rounded-lg text-sm hidden">
                <p class="font-semibold text-gray-600 mb-1">Sources:</p>
                <ul id="sources-ul" class="list-disc list-inside space-y-1 text-gray-500"></ul>
            </div>
        </div>
    </div>

    <script type="module">
        // --- Firebase Global Setup (Required by Canvas Environment) ---
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        // --- Core Application Constants and Elements ---

        const API_URL = "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent";
        const apiKey = ""; // Canvas will provide this if empty

        const outputArea = document.getElementById('output-area');
        const loadingIndicator = document.getElementById('loading-indicator');
        const sourceList = document.getElementById('source-list');
        const sourcesUl = document.getElementById('sources-ul');
        
        // Function defined here for programmatic access
        async function generateCharacter() {
            const generateButton = document.getElementById('generateButton');
            
            // Get inputs
            const charName = document.getElementById('charName').value.trim() || "A nameless protagonist";
            const charGenre = document.getElementById('charGenre').value.trim();
            let charConcept = document.getElementById('charConcept').value.trim(); // Now holds specific details (optional)

            // --- Validation: Only Genre is required ---
            if (!charGenre) {
                outputArea.innerHTML = '<p class="text-red-500 font-semibold">Please fill in the **Genre** field to start forging a character!</p>';
                outputArea.classList.remove('border-solid', 'border-violet-400');
                outputArea.classList.add('border-dashed', 'border-red-400');
                return;
            }
            
            // --- Set Fallback for Optional Details ---
            if (!charConcept) {
                // If the user left details blank, provide instructions for the model to create its own concept.
                charConcept = "No specific details provided. Generate the Core Concept and details based entirely on the Name and Genre, focusing on typical tropes and themes of the genre.";
            }

            // UI State: Loading
            try {
                loadingIndicator.classList.remove('hidden');
                outputArea.innerHTML = '';
                outputArea.classList.remove('border-solid', 'border-violet-400', 'border-red-400');
                outputArea.classList.add('border-dashed', 'border-gray-300');
                generateButton.disabled = true;
                generateButton.innerHTML = '<div class="spinner"></div> Forging...';
                sourceList.classList.add('hidden');
                sourcesUl.innerHTML = '';
            
                // 1. Define the system instruction for the model's persona and output format
                const systemPrompt = `You are a creative writer and character designer specializing in developing unique and compelling original characters (OCs) for fiction. Your task is to take a user's basic concept (Name, Genre, and Specific Details) and expand it into a detailed, ready-to-use character profile. The output must be presented in Markdown with H2 headers (##) for each section.`;

                // 2. Construct the DETAILED and ASSERTIVE user query
                const userQuery = `**STRICTLY ADHERE to the following foundational inputs for the character profile:**
                **Character Name:** ${charName}
                **Primary Genre:** ${charGenre}
                **Foundational Details:** ${charConcept}
    
                Now, generate the complete original character profile. The content must be creative but grounded entirely within the specified **Genre** and the list of **Foundational Details**. The **Name** must be used throughout the profile.

                The profile must follow this structure precisely, using Markdown H2 headers (##) for each section:
                
                ## Character Summary (Logline)
                A single, compelling sentence or two describing **${charName}** and their main conflict within the **${charGenre}** genre.
                
                ## Core Personality
                A bulleted list of 3 to 5 key personality traits and a brief description of their public versus private self.
                
                ## Backstory & Motivation
                A detailed paragraph explaining their origins, what drives them, and the defining event that shaped them.
                
                ## Defining Quirk & Flaw
                A brief description of their most recognizable habit or unique ability, and a critical character flaw that holds them back.
                
                ## Thematic Conflict
                A one-sentence summary of the main internal or external struggle that defines the character's arc.`;

                const payload = {
                    contents: [{ parts: [{ text: userQuery }] }],
                    systemInstruction: { parts: [{ text: systemPrompt }] },
                    tools: [{ "google_search": {} }], 
                };
                
                let finalResult = null;
                let maxRetries = 5;
                let delay = 1000; 

                for (let i = 0; i < maxRetries; i++) {
                    try {
                        const response = await fetch(API_URL, {
                            method: 'POST',
                            headers: { 'Content-Type': 'application/json' },
                            body: JSON.stringify(payload)
                        });

                        if (!response.ok) {
                            if (response.status === 429 && i < maxRetries - 1) {
                                await new Promise(resolve => setTimeout(resolve, delay));
                                delay *= 2; 
                                continue;
                            } else {
                                const errorBody = await response.text();
                                throw new Error(`API Request Failed: ${response.status} - ${errorBody}`);
                            }
                        }

                        finalResult = await response.json();
                        break; 
                    } catch (error) {
                        console.error('Fetch attempt failed:', error);
                        if (i === maxRetries - 1) throw error; 
                        await new Promise(resolve => setTimeout(resolve, delay));
                        delay *= 2; 
                    }
                }

                // --- Processing Results ---

                if (finalResult && finalResult.candidates && finalResult.candidates.length > 0) {
                    const candidate = finalResult.candidates[0];
                    const generatedText = candidate.content?.parts?.[0]?.text || "The model returned an empty response.";

                    // 1. Format and Display the main text
                    outputArea.innerHTML = formatMarkdownToHtml(generatedText);
                    outputArea.classList.remove('border-dashed', 'border-gray-300');
                    outputArea.classList.add('border-solid', 'border-violet-400');

                    // 2. Extract and display grounding sources (citations)
                    let sources = [];
                    const groundingMetadata = candidate.groundingMetadata;

                    if (groundingMetadata && groundingMetadata.groundingAttributions) {
                        sources = groundingMetadata.groundingAttributions
                            .map(attribution => ({
                                uri: attribution.web?.uri,
                                title: attribution.web?.title,
                            }))
                            .filter(source => source.uri && source.title);
                    }

                    if (sources.length > 0) {
                        sourcesUl.innerHTML = '';
                        sources.forEach(source => {
                            const li = document.createElement('li');
                            li.innerHTML = `<a href="${source.uri}" target="_blank" class="text-violet-600 hover:text-violet-800 underline transition duration-150">${source.title}</a>`;
                            sourcesUl.appendChild(li);
                        });
                        sourceList.classList.remove('hidden');
                    }
                } else {
                    outputArea.innerHTML = '<p class="text-red-500 font-semibold">The API call was successful but the model did not return a valid character profile. Please adjust your prompt and try again.</p>';
                    outputArea.classList.remove('border-dashed', 'border-gray-300');
                    outputArea.classList.add('border-solid', 'border-red-400');
                }
            } catch (error) {
                // Catch all errors (network, API, parsing, etc.) and display a message
                console.error("Critical Generation Error:", error);
                outputArea.innerHTML = `<p class="text-red-500 font-semibold">A critical error occurred while contacting the service. Check the browser console for technical details.</p>`;
                outputArea.classList.remove('border-dashed', 'border-gray-300');
                outputArea.classList.add('border-solid', 'border-red-400');
            } finally {
                // Ensure UI is always reset regardless of success or failure
                resetUI();
            }
        }
        
        /**
         * Converts Markdown output to readable HTML.
         */
        function formatMarkdownToHtml(markdownText) {
            if (!markdownText) return '';
            
            let html = markdownText;

            // 1. Replace headers (##) with bolded, colored text and a margin
            html = html.replace(/^##\s*(.*)$/gm, '<h3 class="text-xl font-semibold title-font text-violet-700 mt-4 mb-2">$1</h3>');

            // 2. Replace lists (-) and ensure proper wrapping
            html = html.replace(/^\*\s*(.*)$/gm, '<li>$1</li>');
            html = html.replace(/((?:<li>.*?<\/li>\s*)+)/gs, (match) => {
                if (match.trim().startsWith('<ul')) return match; 
                return `<ul class="list-disc list-inside ml-4 space-y-1">${match.trim()}</ul>`;
            });

            // 3. Replace double newlines with paragraphs
            html = html.replace(/\n\n/g, '</p><p>');

            // 4. Clean up single newlines (treat them as soft breaks)
            html = html.replace(/\n/g, '<br/>');

            // 5. Ensure the whole thing is wrapped in a paragraph if it doesn't start with a known block element
            if (!html.startsWith('<h3') && !html.startsWith('<ul')) {
                html = `<p>${html}</p>`;
            }
            return html;
        }

        /**
         * Resets the UI elements to their default state.
         */
        function resetUI(defaultText = 'Generate Character Profile') {
            const generateButton = document.getElementById('generateButton');
            loadingIndicator.classList.add('hidden');
            generateButton.disabled = false;
            generateButton.innerHTML = defaultText;
        }

        // --- Correct Event Listener Attachment ---
        document.addEventListener('DOMContentLoaded', function() {
            const generateButton = document.getElementById('generateButton');
            if (generateButton) {
                generateButton.addEventListener('click', generateCharacter);
            }
        });
    </script>
</body>
</html>

