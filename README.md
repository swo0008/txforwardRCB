# txforwardRCB
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Rank-Choice Voting</title>
    <!-- Tailwind CSS for modern, responsive styling -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f3f4f6;
            color: #1f2937;
        }
        /* Custom styles for drag and drop list */
        .draggable {
            cursor: grab;
            user-select: none;
        }
        .dragging {
            opacity: 0.5;
        }
        .placeholder {
            height: 4rem;
            background-color: #e5e7eb;
            border-radius: 0.5rem;
            margin-bottom: 0.5rem;
            transition: all 0.2s;
        }
    </style>
</head>
<body class="bg-gray-100 min-h-screen flex items-center justify-center p-4">

    <!-- Main Container -->
    <div class="max-w-4xl w-full bg-white rounded-xl shadow-2xl p-8">
        <h1 class="text-4xl font-extrabold text-center text-indigo-800 mb-2">Rank-Choice Voting</h1>
        <p class="text-center text-sm text-gray-500 mb-6">Sponsored by the Texas Forward Party</p>
        <p class="text-center text-gray-600 mb-8">Create an election or join an existing one to cast your ballot.</p>

        <!-- UI State Management -->
        <div id="create-election" class="space-y-6">
            <h2 class="text-2xl font-bold text-gray-800">Create a New Election</h2>
            <input id="election-question" type="text" placeholder="e.g., 'Favorite Movie of 2025?'" class="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-transparent">
            <textarea id="candidate-list" rows="4" placeholder="Enter candidate names, one per line." class="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-transparent"></textarea>
            <button id="create-btn" class="w-full p-3 bg-indigo-600 text-white font-semibold rounded-lg shadow-md hover:bg-indigo-700 transition-colors">Create Election</button>

            <div class="text-center text-gray-500 text-sm mt-4">
                <p>Or, if you have an Election ID, enter it below to join.</p>
                <input id="join-election-id" type="text" placeholder="Enter Election ID" class="w-full p-3 mt-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-transparent">
                <button id="join-btn" class="w-full p-3 mt-2 bg-gray-500 text-white font-semibold rounded-lg shadow-md hover:bg-gray-600 transition-colors">Join Election</button>
            </div>
        </div>

        <div id="voting-interface" class="hidden">
            <h2 id="election-display-question" class="text-2xl font-bold text-gray-800 mb-4"></h2>
            <div id="vote-container" class="space-y-4">
                <p class="text-gray-600">Drag and drop candidates to rank them from top (1st) to bottom (last).</p>
                <ul id="candidate-list-drag" class="bg-gray-50 p-4 rounded-xl border border-dashed border-gray-300 min-h-48 space-y-2">
                    <!-- Draggable candidates will be populated here -->
                </ul>
                <button id="submit-vote-btn" class="w-full p-3 bg-emerald-600 text-white font-semibold rounded-lg shadow-md hover:bg-emerald-700 transition-colors">Submit My Ballot</button>
            </div>
            <p id="submission-message" class="text-center mt-4 text-green-600 font-semibold hidden"></p>
        </div>

        <div id="results-container" class="hidden">
            <h2 class="text-2xl font-bold text-gray-800 mt-8 mb-4">Election Results</h2>
            <p id="total-ballots-count" class="text-gray-600 mb-2"></p>
            <div id="election-results" class="bg-gray-50 p-4 rounded-xl border border-dashed border-gray-300">
                <!-- Results will be displayed here -->
            </div>
            <p id="winner-message" class="text-2xl font-bold text-center mt-6 text-indigo-700"></p>
            <p id="election-id-display" class="text-center text-sm text-gray-400 mt-4"></p>
            <p id="user-id-display" class="text-center text-xs text-gray-400 mt-1"></p>
        </div>

        <div id="loading-spinner" class="hidden text-center mt-8">
            <div class="spinner border-t-4 border-b-4 border-indigo-500 rounded-full w-12 h-12 animate-spin mx-auto"></div>
            <p class="text-gray-500 mt-2">Loading...</p>
        </div>
    </div>

    <!-- Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInWithCustomToken, signInAnonymously } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, getDoc, setDoc, collection, query, onSnapshot, addDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Global variables provided by the Canvas environment
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        let db, auth;
        let userId;
        let currentElectionId = null;

        // --- Firebase Initialization and Authentication ---
        async function initializeFirebase() {
            try {
                const app = initializeApp(firebaseConfig);
                auth = getAuth(app);
                db = getFirestore(app);

                // Sign in with the provided custom token or anonymously
                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                } else {
                    await signInAnonymously(auth);
                }

                userId = auth.currentUser.uid;
                document.getElementById('user-id-display').textContent = `Your User ID: ${userId}`;

                console.log("Firebase initialized and user authenticated.");
                // Check for election ID in URL and join if it exists
                const urlParams = new URLSearchParams(window.location.search);
                const electionIdFromUrl = urlParams.get('electionId');
                if (electionIdFromUrl) {
                    await joinElection(electionIdFromUrl);
                }

            } catch (error) {
                console.error("Error initializing Firebase:", error);
                // Handle the error gracefully, maybe show a message to the user
            }
        }

        // --- UI Element References ---
        const createElectionSection = document.getElementById('create-election');
        const votingInterfaceSection = document.getElementById('voting-interface');
        const resultsContainer = document.getElementById('results-container');
        const loadingSpinner = document.getElementById('loading-spinner');

        const createBtn = document.getElementById('create-btn');
        const joinBtn = document.getElementById('join-btn');
        const submitVoteBtn = document.getElementById('submit-vote-btn');

        const electionQuestionInput = document.getElementById('election-question');
        const candidateListTextarea = document.getElementById('candidate-list');
        const joinElectionIdInput = document.getElementById('join-election-id');
        const electionDisplayQuestion = document.getElementById('election-display-question');
        const candidateListDrag = document.getElementById('candidate-list-drag');
        const submissionMessage = document.getElementById('submission-message');
        const electionResultsDisplay = document.getElementById('election-results');
        const winnerMessage = document.getElementById('winner-message');
        const totalBallotsCount = document.getElementById('total-ballots-count');
        const electionIdDisplay = document.getElementById('election-id-display');

        // --- Drag and Drop Logic ---
        function setupDraggableList() {
            let dragSrcEl = null;

            function handleDragStart(e) {
                this.classList.add('dragging');
                dragSrcEl = this;
                e.dataTransfer.effectAllowed = 'move';
                e.dataTransfer.setData('text/plain', this.textContent);
            }

            function handleDragOver(e) {
                e.preventDefault(); // Required for drop event
                e.dataTransfer.dropEffect = 'move';
                return false;
            }

            function handleDrop(e) {
                e.stopPropagation();
                if (dragSrcEl !== this) {
                    const droppedIndex = Array.from(this.parentNode.children).indexOf(this);
                    const draggedIndex = Array.from(dragSrcEl.parentNode.children).indexOf(dragSrcEl);
                    if (droppedIndex > draggedIndex) {
                        this.parentNode.insertBefore(dragSrcEl, this.nextSibling);
                    } else {
                        this.parentNode.insertBefore(dragSrcEl, this);
                    }
                }
                return false;
            }

            function handleDragEnd(e) {
                this.classList.remove('dragging');
                updateRankingDisplay();
            }

            const items = candidateListDrag.querySelectorAll('li');
            items.forEach(item => {
                item.addEventListener('dragstart', handleDragStart, false);
                item.addEventListener('dragover', handleDragOver, false);
                item.addEventListener('drop', handleDrop, false);
                item.addEventListener('dragend', handleDragEnd, false);
            });
        }
        
        // --- Helper function to update the ranking numbers on the list items
        function updateRankingDisplay() {
            const items = candidateListDrag.querySelectorAll('li');
            items.forEach((item, index) => {
                const rankSpan = item.querySelector('span');
                rankSpan.textContent = `${index + 1}.`;
            });
        }
        
        // --- Election Creation and Joining ---
        async function createElection() {
            const question = electionQuestionInput.value.trim();
            const candidateNames = candidateListTextarea.value.split('\n').map(name => name.trim()).filter(name => name.length > 0);

            if (!question || candidateNames.length < 2) {
                alert("Please enter a question and at least two candidates.");
                return;
            }
            
            showLoading(true);

            try {
                // Use the user's ID to generate a unique election ID
                const electionId = `${userId}-${Date.now()}`;
                const electionRef = doc(db, 'artifacts', appId, 'public', 'data', 'elections', electionId);

                await setDoc(electionRef, {
                    question: question,
                    candidates: candidateNames,
                    creatorId: userId,
                    createdAt: new Date()
                });

                showLoading(false);
                await joinElection(electionId);
            } catch (error) {
                console.error("Error creating election:", error);
                showLoading(false);
                alert("Failed to create election. Please try again.");
            }
        }

        async function joinElection(electionId) {
            showLoading(true);
            const electionRef = doc(db, 'artifacts', appId, 'public', 'data', 'elections', electionId);
            try {
                const electionSnap = await getDoc(electionRef);
                if (!electionSnap.exists()) {
                    throw new Error("Election not found.");
                }

                const electionData = electionSnap.data();
                currentElectionId = electionId;
                
                // Update UI for voting
                electionDisplayQuestion.textContent = electionData.question;
                createElectionSection.classList.add('hidden');
                votingInterfaceSection.classList.remove('hidden');
                resultsContainer.classList.remove('hidden');

                // Populate draggable candidate list
                candidateListDrag.innerHTML = '';
                electionData.candidates.forEach((name, index) => {
                    const li = document.createElement('li');
                    li.textContent = name;
                    li.classList.add('draggable', 'p-3', 'bg-white', 'rounded-lg', 'shadow', 'flex', 'items-center', 'font-semibold', 'text-gray-700');
                    li.setAttribute('draggable', 'true');
                    li.innerHTML = `<span class="font-bold text-lg text-indigo-500 mr-2">${index + 1}.</span>${name}`;
                    candidateListDrag.appendChild(li);
                });
                setupDraggableList();

                // Display shareable link and election ID
                const shareableLink = `${window.location.origin}${window.location.pathname}?electionId=${currentElectionId}`;
                electionIdDisplay.innerHTML = `Share this Election: <br><a href="${shareableLink}" class="text-indigo-500 hover:underline break-all">${shareableLink}</a>`;
                
                // Listen for real-time ballot updates
                listenForBallots(electionId, electionData.candidates);
                
                showLoading(false);
                
            } catch (error) {
                console.error("Error joining election:", error);
                showLoading(false);
                alert("Failed to join election. Please check the ID and try again.");
            }
        }

        // --- Ballot Submission ---
        async function submitBallot() {
            const rankedChoices = Array.from(candidateListDrag.children).map(li => li.textContent.replace(/^\d+\.\s*/, '').trim());

            try {
                const ballotsRef = collection(db, 'artifacts', appId, 'public', 'data', 'elections', currentElectionId, 'ballots');
                await addDoc(ballotsRef, {
                    choices: rankedChoices,
                    voterId: userId,
                    timestamp: new Date()
                });
                
                submissionMessage.textContent = "Your ballot has been submitted!";
                submissionMessage.classList.remove('hidden');
                submitVoteBtn.disabled = true; // Disable button after submission
                submitVoteBtn.textContent = 'Ballot Submitted!';

            } catch (error) {
                console.error("Error submitting ballot:", error);
                alert("Failed to submit ballot. Please try again.");
            }
        }
        
        // --- Real-time Results Listener ---
        function listenForBallots(electionId, candidates) {
            const ballotsRef = collection(db, 'artifacts', appId, 'public', 'data', 'elections', electionId, 'ballots');
            
            onSnapshot(ballotsRef, (snapshot) => {
                const allBallots = [];
                snapshot.forEach(doc => {
                    allBallots.push(doc.data().choices);
                });
                
                // Check if any ballots exist before running election logic
                if (allBallots.length > 0) {
                    totalBallotsCount.textContent = `Total ballots submitted: ${allBallots.length}`;
                    const winner = runIrVotedElection(allBallots, candidates);
                    winnerMessage.textContent = `The final winner is: ${winner}`;
                } else {
                    totalBallotsCount.textContent = "No ballots submitted yet.";
                    electionResultsDisplay.innerHTML = '';
                    winnerMessage.textContent = '';
                }
            });
        }
        
        // --- IRV Algorithm (ported from Python) ---
        function runIrVotedElection(ballots, candidates) {
            let activeCandidates = [...candidates];
            let roundNumber = 1;

            let resultHtml = '<h3>Election Progress:</h3>';

            while (true) {
                resultHtml += `<div class="mt-4"><h4 class="font-semibold text-lg">--- Round ${roundNumber} ---</h4>`;
                let voteCounts = {};
                activeCandidates.forEach(c => voteCounts[c] = 0);

                ballots.forEach(ballot => {
                    for (const choice of ballot) {
                        if (activeCandidates.includes(choice)) {
                            voteCounts[choice]++;
                            break;
                        }
                    }
                });

                resultHtml += `<p class="font-medium mt-2">Current Tally:</p>`;
                const sortedTally = Object.entries(voteCounts).sort(([, a], [, b]) => b - a);
                sortedTally.forEach(([candidate, votes]) => {
                    resultHtml += `<p class="ml-4">- ${candidate}: ${votes} votes</p>`;
                });

                const totalVotes = ballots.length;
                const majorityThreshold = Math.floor(totalVotes / 2) + 1;

                // Check for a winner
                for (const [candidate, votes] of sortedTally) {
                    if (votes >= majorityThreshold) {
                        electionResultsDisplay.innerHTML = resultHtml;
                        return candidate;
                    }
                }

                // No winner, eliminate a candidate
                if (activeCandidates.length <= 1) {
                    electionResultsDisplay.innerHTML = resultHtml;
                    return activeCandidates.length > 0 ? activeCandidates[0] : "No winner";
                }

                const minVotes = Math.min(...Object.values(voteCounts));
                const losers = sortedTally.filter(([, votes]) => votes === minVotes).map(([candidate]) => candidate);
                
                const candidateToEliminate = losers[0]; // Simple tie-breaker
                
                resultHtml += `<p class="mt-2 text-red-600">Eliminating candidate: ${candidateToEliminate} with ${minVotes} votes.</p></div>`;
                activeCandidates = activeCandidates.filter(c => c !== candidateToEliminate);
                roundNumber++;
            }
        }
        
        // --- UI & Event Handlers ---
        function showLoading(show) {
            loadingSpinner.classList.toggle('hidden', !show);
            document.body.classList.toggle('pointer-events-none', show);
            document.body.classList.toggle('opacity-50', show);
        }

        createBtn.addEventListener('click', createElection);
        joinBtn.addEventListener('click', () => joinElection(joinElectionIdInput.value.trim()));
        submitVoteBtn.addEventListener('click', submitBallot);
        
        // Initialize the application
        window.addEventListener('load', initializeFirebase);
    </script>
</body>
</html>
