# codtech-task3
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Real-time Collaboration Tool</title>
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Inter font -->
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f3f4f6;
            display: flex;
            justify-content: center;
            align-items: flex-start; /* Align to top for better content display */
            min-height: 100vh;
            padding: 2rem;
            box-sizing: border-box;
        }
        .container {
            background-color: #ffffff;
            border-radius: 1rem;
            box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
            width: 100%;
            max-width: 1000px;
            display: flex;
            flex-direction: column;
            overflow: hidden;
        }
        @media (min-width: 768px) {
            .container {
                flex-direction: row;
            }
        }
        textarea {
            resize: vertical; /* Allow vertical resizing */
            min-height: 300px; /* Minimum height for textarea */
            max-height: 80vh; /* Max height to prevent overflow on small screens */
        }
        .message-box {
            position: fixed;
            top: 20px;
            left: 50%;
            transform: translateX(-50%);
            background-color: #4CAF50;
            color: white;
            padding: 15px 20px;
            border-radius: 8px;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
            z-index: 1000;
            opacity: 0;
            transition: opacity 0.5s ease-in-out;
        }
        .message-box.show {
            opacity: 1;
        }
    </style>
</head>
<body class="antialiased">
    <div class="container">
        <!-- Left Panel: Editor -->
        <div class="flex-1 p-6 md:p-8 border-b md:border-b-0 md:border-r border-gray-200">
            <h1 class="text-3xl font-bold text-gray-800 mb-4">Collaborative Editor</h1>
            <p class="text-sm text-gray-600 mb-6">Edit this text in real-time with others!</p>

            <div id="loading-indicator" class="text-center py-8 text-gray-500 hidden">
                <div class="animate-spin rounded-full h-12 w-12 border-b-2 border-gray-900 mx-auto mb-4"></div>
                <p>Loading collaboration data...</p>
            </div>

            <textarea
                id="editor"
                class="w-full p-4 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 transition duration-200 text-gray-800 text-base"
                placeholder="Start typing here..."
                rows="15"
            ></textarea>

            <div class="mt-4 text-sm text-gray-700">
                Your User ID: <span id="user-id" class="font-semibold text-blue-600">Loading...</span>
            </div>
        </div>

        <!-- Right Panel: Active Users -->
        <div class="w-full md:w-64 p-6 md:p-8 bg-gray-50">
            <h2 class="text-xl font-semibold text-gray-800 mb-4">Active Users</h2>
            <ul id="active-users-list" class="space-y-2 text-gray-700">
                <li class="text-gray-500">No active users yet.</li>
            </ul>
        </div>
    </div>

    <!-- Custom Message Box -->
    <div id="message-box" class="message-box"></div>

    <!-- Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, setDoc, onSnapshot, collection, query, where, serverTimestamp, deleteDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Global variables provided by the Canvas environment
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        let app;
        let db;
        let auth;
        let userId = null;
        let isAuthReady = false;

        const editor = document.getElementById('editor');
        const userIdSpan = document.getElementById('user-id');
        const activeUsersList = document.getElementById('active-users-list');
        const loadingIndicator = document.getElementById('loading-indicator');
        const messageBox = document.getElementById('message-box');

        // Function to show custom message box
        function showMessage(message, type = 'success') {
            messageBox.textContent = message;
            messageBox.className = `message-box show ${type === 'error' ? 'bg-red-500' : 'bg-green-500'}`;
            setTimeout(() => {
                messageBox.className = 'message-box';
            }, 3000); // Hide after 3 seconds
        }

        // Initialize Firebase and authenticate
        async function initializeFirebase() {
            try {
                if (Object.keys(firebaseConfig).length === 0) {
                    console.error("Firebase config is empty. Please ensure __firebase_config is properly set.");
                    showMessage("Firebase configuration missing. Cannot connect.", "error");
                    return;
                }
                app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);

                // Show loading indicator
                loadingIndicator.classList.remove('hidden');
                editor.disabled = true;

                // Authenticate user
                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                } else {
                    await signInAnonymously(auth);
                }

                // Listen for auth state changes to get the user ID
                onAuthStateChanged(auth, (user) => {
                    if (user) {
                        userId = user.uid;
                        userIdSpan.textContent = userId;
                        isAuthReady = true;
                        console.log("User authenticated:", userId);
                        setupRealtimeListeners(); // Setup listeners after auth is ready
                        editor.disabled = false; // Enable editor after auth and data load
                        loadingIndicator.classList.add('hidden'); // Hide loading indicator
                    } else {
                        console.log("No user signed in.");
                        userIdSpan.textContent = "Not authenticated";
                        isAuthReady = false;
                        editor.disabled = true; // Keep editor disabled if not authenticated
                        loadingIndicator.classList.add('hidden'); // Hide loading indicator
                    }
                });

            } catch (error) {
                console.error("Error initializing Firebase or authenticating:", error);
                showMessage(`Error: ${error.message}`, "error");
                loadingIndicator.classList.add('hidden');
                editor.disabled = true;
            }
        }

        // Firestore paths
        const collaborationDocRef = (db) => doc(db, `artifacts/${appId}/public/data/collaboration/document`);
        const presenceCollectionRef = (db) => collection(db, `artifacts/${appId}/public/data/presence`);
        const userPresenceDocRef = (db, uid) => doc(db, `artifacts/${appId}/public/data/presence/${uid}`);

        // Debounce function to limit Firestore writes
        let debounceTimeout;
        function debounce(func, delay) {
            return function(...args) {
                clearTimeout(debounceTimeout);
                debounceTimeout = setTimeout(() => func.apply(this, args), delay);
            };
        }

        // Update text in Firestore
        const updateTextInFirestore = debounce(async (text) => {
            if (!db || !userId) {
                console.warn("Firestore not initialized or user not authenticated. Cannot update text.");
                return;
            }
            try {
                await setDoc(collaborationDocRef(db), { content: text }, { merge: true });
                // console.log("Document updated successfully!");
            } catch (error) {
                console.error("Error updating document:", error);
                showMessage("Failed to save changes.", "error");
            }
        }, 500); // Debounce by 500ms

        // Update user presence in Firestore
        const updatePresence = async () => {
            if (!db || !userId) return;
            try {
                await setDoc(userPresenceDocRef(db, userId), {
                    userId: userId,
                    lastActivity: serverTimestamp(),
                    status: 'online'
                }, { merge: true });

                // Set up onDisconnect to remove presence when user goes offline
                // This is a client-side feature and works best when the client explicitly closes or loses connection.
                // For more robust presence, consider server-side solutions or short TTLs.
                const presenceRef = userPresenceDocRef(db, userId);
                const { set, update, delete: deleteField } = await import("https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js");
                await setDoc(presenceRef, { status: 'offline', lastActivity: serverTimestamp() }, { merge: true }); // Initial set to establish connection
                // No direct onDisconnect for Firestore documents in web SDK, this would require Cloud Functions.
                // The current implementation relies on periodic updates and filtering by lastActivity.
            } catch (error) {
                console.error("Error updating presence:", error);
            }
        };

        // Setup real-time listeners for collaboration document and active users
        function setupRealtimeListeners() {
            if (!db || !isAuthReady) {
                console.warn("Firestore or Auth not ready. Skipping real-time listeners setup.");
                return;
            }

            // Listen for changes to the collaborative document
            onSnapshot(collaborationDocRef(db), (docSnap) => {
                if (docSnap.exists()) {
                    const data = docSnap.data();
                    const currentText = editor.value;
                    const newText = data.content || '';

                    // Only update if the text from Firestore is different from current editor value
                    // This prevents cursor jumping when the user is typing
                    if (currentText !== newText) {
                        const selectionStart = editor.selectionStart;
                        const selectionEnd = editor.selectionEnd;
                        editor.value = newText;
                        // Restore cursor position if possible
                        if (document.activeElement === editor) {
                            editor.setSelectionRange(selectionStart, selectionEnd);
                        }
                    }
                } else {
                    editor.value = ''; // Document doesn't exist, clear editor
                }
            }, (error) => {
                console.error("Error listening to document:", error);
                showMessage("Error loading document data.", "error");
            });

            // Listen for changes to active users
            onSnapshot(presenceCollectionRef(db), (snapshot) => {
                activeUsersList.innerHTML = ''; // Clear existing list
                let hasActiveUsers = false;
                const now = Date.now();
                snapshot.forEach((doc) => {
                    const userData = doc.data();
                    // Consider a user active if their last activity was within the last 10 seconds
                    const lastActivityTime = userData.lastActivity ? userData.lastActivity.toDate().getTime() : 0;
                    if (userData.userId && (now - lastActivityTime < 10000 || userData.status === 'online')) {
                        const listItem = document.createElement('li');
                        listItem.className = 'flex items-center space-x-2';
                        listItem.innerHTML = `
                            <span class="relative flex h-3 w-3">
                                <span class="animate-ping absolute inline-flex h-full w-full rounded-full bg-green-400 opacity-75"></span>
                                <span class="relative inline-flex rounded-full h-3 w-3 bg-green-500"></span>
                            </span>
                            <span>${userData.userId === userId ? 'You (' + userData.userId + ')' : userData.userId}</span>
                        `;
                        activeUsersList.appendChild(listItem);
                        hasActiveUsers = true;
                    }
                });

                if (!hasActiveUsers) {
                    const noUsersItem = document.createElement('li');
                    noUsersItem.className = 'text-gray-500';
                    noUsersItem.textContent = 'No active users yet.';
                    activeUsersList.appendChild(noUsersItem);
                }
            }, (error) => {
                console.error("Error listening to active users:", error);
            });

            // Periodically update presence
            setInterval(updatePresence, 5000); // Update every 5 seconds
            updatePresence(); // Initial presence update on load

            // Handle user typing
            editor.addEventListener('input', (e) => {
                updateTextInFirestore(e.target.value);
                updatePresence(); // Also update presence when typing
            });

            // Handle window beforeunload to mark user offline (best effort)
            window.addEventListener('beforeunload', async () => {
                if (db && userId) {
                    try {
                        // This might not always complete, but it's a good attempt
                        await setDoc(userPresenceDocRef(db, userId), { status: 'offline', lastActivity: serverTimestamp() }, { merge: true });
                    } catch (e) {
                        console.warn("Failed to update presence to offline on unload:", e);
                    }
                }
            });
        }

        // Initialize everything when the window loads
        window.onload = initializeFirebase;
    </script>
</body>
</html>
