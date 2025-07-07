# Pokémon TCG Hand Simulator - High-Level Specifications

## 1. Overview

A web application that simulates drawing an opening hand in the Pokémon Trading Card Game. The primary goal is to provide a simple way for players to test their deck's consistency by checking how often they get a playable starting hand (i.e., a hand with at least one Basic Pokémon).

The key differentiator from existing tools is the input method: the user will copy-paste their decklist directly from the Pokémon TCG Live game client, without needing to manually build the deck in the app or upload files.

## 2. Core Features (Version 1.0)

*   **Deck Input:** A single text area where users can paste a decklist.
*   **Format Support:** The app will parse decklists in the standard Pokémon TCG Live export format.
*   **Deck Validation:**
    *   Verify the deck contains exactly 60 cards.
    *   The parser must validate that the decklist contains "Pokémon:", "Trainer:", and "Energy:" sections.
*   **Hand Simulation:**
    *   On user request, simulate drawing a 7-card starting hand.
    *   Correctly implement the official mulligan rule: if the hand contains no Basic Pokémon, it is reshuffled and a new hand is drawn.
*   **Result Display:**
    *   Display the images of the 7 cards in the final simulated hand.
    *   Display the number of mulligans that were required to get a valid hand.

## 3. User Flow

1.  The user visits the site and sees a clean interface with a large text area for their decklist and a "Simulate Hand" button.
2.  The user pastes their 60-card decklist from TCG Live into the text area.
3.  The user clicks "Simulate Hand".
4.  The frontend displays a loading indicator while it sends the decklist to the backend API.
5.  The backend processes the decklist:
    *   **If valid:** It runs the simulation and returns the result. The frontend then displays the images for the 7 cards in the final hand, along with the mulligan count.
    *   **If invalid (e.g., wrong card count, unrecognized format):** It returns a clear error message, which the frontend displays to the user.

## 4. Technical Specifications

### Frontend
*   **Framework:** React.js
*   **Components:**
    *   `DeckInput`: A component with a `<textarea>` for the decklist.
    *   `SimulationControl`: A button to trigger the simulation.
    *   `ResultDisplay`: A component to show the simulated hand (with card images) and mulligan count.
    *   `ErrorMessage`: A component to display validation errors.
    *   `LoadingIndicator`: A component to show while the simulation is in progress.
*   **Functionality:**
    *   Client-side state management for the decklist string, loading state, results, and any errors.
    *   API call to the backend for simulation.
    *   Display of results or errors returned from the backend.

### Backend
*   **Framework:** Flask (Python)
*   **API Endpoint:** `POST /api/simulate`
    *   **Request Body:**
        ```json
        {
          "decklist": "Pokémon: 10\n1 Latias ex SSP 76\n..."
        }
        ```
    *   **Success Response (200 OK):**
        ```json
        {
          "hand": [
            // This array will contain 7 card objects.
            {
              "name": "Iono",
              "type": "Trainer",
              "identifier": "PAL 185",
              "imageUrl": "https://limitlesstcg.com/cards/PAL/185"
            },
            {
              "name": "Koraidon ex",
              "type": "Pokémon",
              "identifier": "SVI 125",
              "imageUrl": "https://limitlesstcg.com/cards/SVI/125"
            }
            // ... and 5 more cards
          ],
          "mulligans": 0,
          "message": "Simulation successful."
        }
        ```
    *   **Error Response (400 Bad Request):**
        ```json
        {
          "error": "Deck must contain exactly 60 cards. Found 59."
        }
        ```
*   **Core Logic:**
    *   **Deck Parser:** A module to parse the TCG Live decklist format. It will extract card names, quantities, and identifiers (e.g., "PAL 185"), and categorize them (Pokémon, Trainer, Energy). The parser must handle the following format, including the "Pokémon", "Trainer", "Energy", and "Total Cards" lines.
        ```
        Pokémon: 10
        1 Latias ex SSP 76
        2 Koraidon ex SVI 125
        
        Trainer: 14
        4 Ultra Ball SVI 196
        3 Professor Sada's Vitality PAR 170
        
        Energy: 2
        8 Basic {F} Energy Energy 14
        
        Total Cards: 60
        ```
    *   **Card Identifier (Is Basic?):** To determine if a Pokémon is a "Basic Pokémon" (and thus valid for a starting hand), the backend will query the `https://pokemontcg.io/v2/cards` API.
        *   The query will use the card name (e.g., `q=name:"Koraidon ex"`).
        *   The API response will be checked for a `subtypes` array containing the value `"Basic"`.
        *   Example: A search for "Pikachu" might return a card object with `"subtypes": ["Basic"]`. A search for "Raichu" might return `"subtypes": ["Stage 1"]`.
        *   API results **must** be cached (e.g., in memory) to avoid repeated queries for the same card name.
    *   **Card Image Scraper:**
        *   For each unique card in the deck, the backend will construct a search URL for `limitlesstcg.com` using the card name and identifier (e.g., `https://limitlesstcg.com/cards?q=Iono+PAL+185`).
        *   It will then fetch the page and extract the image source URL. The target is the `src` attribute of the `<img>` tag located at the XPath `/html/body/main/div/section/div/a/img`.
        *   Image URLs **must** be cached (e.g., in memory or a simple file-based cache) to avoid repeated scraping for the same card.
    *   **Simulator:** A module that takes the parsed deck, creates a list of 60 card objects, shuffles it, and performs the draw/mulligan simulation logic according to official rules (reshuffle and redraw a 7-card hand if no Basic Pokémon are present).

*   **Testing Strategy:**
    *   **Framework:** We will use `pytest` for writing and running automated tests.
    *   **Mocking:** Network requests to `pokemontcg.io` and `limitlesstcg.com` will be mocked using a library like `requests-mock` to ensure tests are fast and deterministic.
    *   **Unit Tests:** Each backend module will have dedicated unit tests.
        *   **Deck Parser:** Tests will verify that various TCG Live export formats are parsed correctly into a structured data format. This includes testing with invalid/malformed strings and incorrect card counts.
        *   **Card Image Scraper:** The scraper will be tested by mocking HTTP requests to `limitlesstcg.com`. We will use sample HTML content to verify that the XPath logic correctly extracts image URLs.
        *   **Simulator:** The randomization logic will be tested by using a fixed seed for the random number generator, ensuring that the deck shuffling and card drawing are repeatable for tests. We will have specific test cases for decks that should and should not trigger a mulligan.
    *   **Integration Tests:** We will use Flask's test client to send requests to the `/api/simulate` endpoint. These tests will cover the full flow from receiving a decklist to returning a simulated hand, asserting the correctness of the HTTP response status and JSON payload.
    *   **Execution:** All tests can be run from the command line, enabling verification of all backend logic without requiring the frontend to be running.

### Database
*   No database will be used in the initial version. The application will be stateless.

## 5. Out of Scope for Version 1.0

*   User accounts, login, or saving decks.
*   Advanced Monte Carlo simulations to calculate probabilities over many trials.
*   Simulating turns beyond the initial hand draw.
*   Support for decklist formats other than the TCG Live export.
*   A visual deck builder. 