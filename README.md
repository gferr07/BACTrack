# BACTrack
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BAC Tracker</title>
    <link rel="manifest" href="manifest.json">
    <script>
        if ('serviceWorker' in navigator) {
            navigator.serviceWorker.register('/service-worker.js')
                .then(reg => console.log('Service Worker Registered', reg))
                .catch(err => console.log('Service Worker Registration Failed', err));
        }

        document.addEventListener("DOMContentLoaded", () => {
            initializeDB();

            document.addEventListener("submit", event => {
                event.preventDefault();
                const form = event.target;
                if (form.id === "addCustomDrinkForm") {
                    const type = document.getElementById("customDrinkType").value;
                    const size = parseFloat(document.getElementById("customDrinkSize").value);
                    const alcohol = parseFloat(document.getElementById("customDrinkABV").value) / 100;
                    saveItem("customDrinks", { type, size, alcohol }, "customDrinkList", displayCustomDrinkItem);
                } else if (form.id === "logDrinkForm") {
                    const type = document.getElementById("drinkType").value;
                    const size = parseFloat(document.getElementById("drinkSize").value);
                    const alcohol = parseFloat(document.getElementById("drinkABV").value) / 100;
                    const drunkness = parseInt(document.getElementById("drunkness").value);
                    saveItem("drinks", { type, size, alcohol, drunkness, time: Date.now() }, "drinkList", displayDrinkItem);
                }
            });
        });

        let db;

        function initializeDB() {
            const request = indexedDB.open("BACTrackerDB", 1);

            request.onupgradeneeded = event => {
                db = event.target.result;
                if (!db.objectStoreNames.contains("drinks")) {
                    db.createObjectStore("drinks", { keyPath: "id", autoIncrement: true });
                }
                if (!db.objectStoreNames.contains("customDrinks")) {
                    db.createObjectStore("customDrinks", { keyPath: "id", autoIncrement: true });
                }
            };

            request.onsuccess = event => {
                db = event.target.result;
                displayItems("drinks", "drinkList", displayDrinkItem);
                displayItems("customDrinks", "customDrinkList", displayCustomDrinkItem);
                calculateBAC(); // Calculate BAC after loading drinks
            };
        }

        function displayItems(storeName, elementId, displayFunction) {
            if (!db) return;
            const transaction = db.transaction(storeName, "readonly");
            const store = transaction.objectStore(storeName);
            const request = store.getAll();
            request.onsuccess = () => {
                document.getElementById(elementId).innerHTML = request.result.map(displayFunction).join("");
            };
        }

        function displayDrinkItem(drink) {
            return `<li>${drink.type} - ${drink.size}oz (${(drink.alcohol * 100).toFixed(1)}%) - Drunkness: ${drink.drunkness}/10 
                    <button onclick="removeItem('drinks', ${drink.id}, 'drinkList', displayDrinkItem)">Remove</button></li>`;
        }

        function displayCustomDrinkItem(drink) {
            return `<div>
                        <button onclick="promptQuickDrink('${drink.type}', ${drink.size}, ${drink.alcohol})">${drink.type}</button>
                        <button onclick="removeItem('customDrinks', ${drink.id}, 'customDrinkList', displayCustomDrinkItem)">Remove</button>
                    </div>`;
        }

        function promptQuickDrink(type, size, alcohol) {
            const drunkness = parseInt(prompt("Enter your drunkness level (1-10):", "5"));
            if (drunkness >= 1 && drunkness <= 10) {
                saveItem("drinks", { type, size, alcohol, drunkness, time: Date.now() }, "drinkList", displayDrinkItem);
            } else {
                alert("Invalid drunkness level. Please enter a number between 1 and 10.");
            }
        }

        function saveItem(storeName, item, elementId, displayFunction) {
            if (!db) return;
            const transaction = db.transaction(storeName, "readwrite");
            const store = transaction.objectStore(storeName);
            store.add(item);
            displayItems(storeName, elementId, displayFunction);
            if (storeName === "drinks") calculateBAC();
        }

        function removeItem(storeName, id, elementId, displayFunction) {
            if (!db) return;
            const transaction = db.transaction(storeName, "readwrite");
            const store = transaction.objectStore(storeName);
            store.delete(id);
            displayItems(storeName, elementId, displayFunction);
            if (storeName === "drinks") calculateBAC();
        }

        function calculateBAC() {
            if (!db) return;
            const transaction = db.transaction("drinks", "readonly");
            const store = transaction.objectStore("drinks");
            const request = store.getAll();
            request.onsuccess = () => {
                let drinks = request.result;
                let totalAlcohol = drinks.reduce((sum, drink) => sum + (drink.size * drink.alcohol * 0.789), 0);
                let elapsedTime = (Date.now() - (drinks[0]?.time || Date.now())) / 3600000;
                let bac = Math.max(((totalAlcohol * 5.14) / (170 * 0.58)) - (0.015 * elapsedTime), 0).toFixed(3);
                document.getElementById("bacLevel").textContent = bac;
                document.getElementById("timeToSoberUp").textContent = `${(bac / 0.015).toFixed(1)} hours`;
            };
        }
    </script>
</head>
<body>
    <h1>Drink Logger</h1>
    <h2>Quick Add Drinks:</h2>
    <div id="customDrinkList"></div>
    <h3>Add Custom Drink:</h3>
    <form id="addCustomDrinkForm">
        <input type="text" id="customDrinkType" placeholder="Drink Type" required>
        <input type="number" id="customDrinkSize" step="0.1" min="0" placeholder="Size (oz)" required>
        <input type="number" id="customDrinkABV" step="0.01" min="0" placeholder="Alcohol % (e.g. 5.5)" required>
        <button type="submit">Add</button>
    </form>
    <h2>Drinks Consumed:</h2>
    <ul id="drinkList"></ul>
    <h2>Estimated BAC: <span id="bacLevel">0.000</span></h2>
    <h2>Time to Sober Up: <span id="timeToSoberUp">0 hours</span></h2>
    <h3>Log a Drink:</h3>
    <form id="logDrinkForm">
        <input type="text" id="drinkType" placeholder="Drink Type" required>
        <input type="number" id="drinkSize" step="0.1" min="0" placeholder="Size (oz)" required>
        <input type="number" id="drinkABV" step="0.01" min="0" placeholder="Alcohol % (e.g. 5.5)" required>
        <label for="drunkness">Drunkness (1-10):</label>
        <input type="number" id="drunkness" min="1" max="10" required>
        <button type="submit">Log Drink</button>
    </form>
</body>
</html>
