## Building an AI-Powered Text Adventure with Google Gemini 

### A deep dive into how I built a text-based adventure game using Spring Boot and Angular, powered by Google Gemini

Welcome to this document about how I created an **AI-driven text adventure game** using **Spring Boot**, **Angular**, and **Google Gemini**. I'll cover the architecture, implementation, and key features of the game in this page.

## 🎮 Overview

This interactive storytelling game presents players with choices that influence the storyline. The backend generates narrative content dynamically using **Google Gemini AI**.

## 🏗️ Architecture

- **Frontend:** Angular + Angular Material ([UI Repo](https://github.com/pratikIT95/text-game-llm-ui))
- **Backend:** Spring Boot REST API ([API Repo](https://github.com/pratikIT95/text-game-llm-api))
- **AI:** Google Gemini API for story generation
- **Data Storage:** Session-based tracking of user choices

## 🚀 How to Start

![The game in action!](game-gif.gif)

### **1️⃣ Run the Backend (Spring Boot)**

To run the backend, you need to generate a Gemini API key and configure it.

#### **Step 1: Get a Gemini API Key**
1. Visit the [Google AI Gemini API](https://ai.google.dev/) page.
2. Sign in with your Google account.
3. Navigate to the API keys section and generate a new API key.
4. Copy the API key for later use.

#### **Step 2: Add the API Key to Your Properties File**
1. Open `text-game-llm-api/src/main/resources/application-local.properties`.
2. Add the following line, replacing `YOUR_API_KEY_HERE` with your actual API key:

```properties
gemini.api.key=YOUR_API_KEY_HERE
```

#### **Step 3: Run the Backend**

```sh
cd text-game-llm-api  
./gradlew bootRun  
```

## 🔧 Backend Implementation (Spring Boot)

### **Conversation Management in the Backend**

One of the critical aspects of the game is maintaining conversation context. Since the Gemini API does not inherently track ongoing conversations, the backend handles it using a **ConcurrentHashMap**, mapping each user to their respective game state.

- Each request maintains its history in the request body itself.
- User inputs are labeled as `user` role, and AI responses are labeled as `model` role.
- The conversation history is stored and updated with each API call.
- This allows Gemini to generate context-aware responses without needing additional storage.

The method `addDataToRequestWithRole` is used to append responses and maintain state:

```java
private GeminiRequest addDataToRequestWithRole(String role, GeminiRequest requestBody, String data) {
    List<Content> contents = requestBody.getContents();
    List<Content> newContents = new ArrayList<>();
    if (contents == null) {
        contents = new ArrayList<>();
    }
    newContents.addAll(contents);
    newContents.add(getContent(role, data));
    requestBody.setContents(newContents);
    return requestBody;
}
```

### **Tuning the Gemini API with a System Instruction**

To guide the AI in generating structured and engaging game content, I provided a **system instruction** that defines the game's lore, character roles, setting, and constraints. The instruction ensures:

- The story follows a **murder mystery theme** set in **Colonial India**.
- The AI randomly assigns a detective to the player from famous literary detectives.
- The game maintains **three structured acts**: introduction, investigation, and resolution.
- Choices affect storytelling but do not guarantee solving the mystery.
- The game ends within a **maximum of 20 prompts**, preventing endless loops.
- The response is strictly formatted as JSON to ease frontend processing.

Example system instruction snippet:

```java
"You are a text adventure game and this is the lore - a murder mystery detective story in early 20th Century, Colonial India. You randomly assign a detective character to the player - Sherlock Holmes with Dr. John Watson, Byomkesh Bakshi with Ajit, Feluda with Topshe, or Hercule Poirot... Your responses should be in pure JSON format with the following fields - storyText, choices (list of 3), isEnding (whether the story has ended), without any markdown tags."
```

## 🎨 Frontend Implementation (Angular)

The **Angular frontend** is responsible for displaying the game and handling user choices. It uses **Angular Material** for styling. The frontend also generates a UUID per every browser session, which is necessary to keep track of states from the backend.


### **Angular Material UI Components**

The frontend uses Angular Material for a smooth and modern UI. The following components are integrated:

- **MatCard**: To display story text.
- **MatButton**: For selecting choices.
- **MatProgressSpinner**: For loading state.

Example UI snippet:

```html
<div class="story-container">
  @if (isLoading) {
    <div class="overlay">
      <mat-spinner class="spinner"></mat-spinner>
    </div>
  }
  @if (storyResponse) {
    <mat-card class="story-card">
      <mat-card-content>
        <p class="story-text">{{ storyResponse.storyText }}</p>
      </mat-card-content>
    </mat-card>
    <div class="choices-container">
      @for (choice of choices; track choice) {
        <button mat-raised-button color="primary" class="choice-button" (click)="makeChoice(choice)">
          {{ choice }}
        </button>
      }
    </div>
  }
  @if (!isStoryStarted && !isStoryEnded) {
    <button mat-raised-button color="accent" class="action-button" (click)="startGame()">
      Start Game
    </button>
  }
  @if (isStoryEnded) {
    <button mat-raised-button color="accent" class="action-button" (click)="startGame()">
      <i class="fa-solid fa-rotate-right"></i>
      Restart Game
    </button>
  }
</div>
```

### **Generating a UUID in Angular**

Each session generates a unique UUID to track the game state. This is done at the app component level where on start of the component, a UUID is generated and stored in sessionStorage. If the browser is refreshed, the state is lost, but this is intentional since a user might prefer to start afresh if they are not finding the story interesting enough.

```typescript
export class AppComponent implements OnInit,OnDestroy {
  ngOnDestroy(): void {
    sessionStorage.removeItem('sessionUuid');
  }
  ngOnInit(): void {
    sessionStorage.setItem('sessionUuid', uuid.v4());
  }
}
```

### **GameService.ts**

```typscript
export class StoryService {

  private baseUrl = 'http://localhost:8080'; // Replace with your backend URL
  private sessionUuid : string = '';

  constructor(private http: HttpClient) {
   }

  startGame(): Observable<any> {
    return this.http.post(`${this.baseUrl}/start/${sessionStorage.getItem('sessionUuid')}`, {});
  }

  continueGame(prompt: string): Observable<any> {
    return this.http.post(`${this.baseUrl}/prompt/${sessionStorage.getItem('sessionUuid')}`, prompt, { headers: { 'Content-Type': 'text/xml' } });
  }
}

```
The GameService is responsible for making API calls to the backend and sending the current UUID as the userId.



### **GameViewComponent.ts**

```typescript
@Component({
  selector: 'app-game-view',
  templateUrl: './game-view.component.html',
  providers: [StoryService],
})
export class GameViewComponent {
  storyResponse: any;
  choices: string[] = [];
  isStoryStarted = false;
  isStoryEnded = false;
  isLoading = false;

  constructor(private storyService: StoryService) { }

  startGame() {
    this.isLoading = true;
    this.storyService.startGame().subscribe(response => {
      this.isStoryStarted = true;
      this.storyResponse = response;
      this.choices = response.choices;
      this.isStoryEnded = response.isEnding;
      this.isLoading = false;
    });
  }
}
```

This component interacts with `StoryService` to update the UI with new game states.

## 🎯 Conclusion

This project demonstrates how **Google Gemini AI** can enhance text-based games by generating **dynamic storytelling experiences**. Future improvements may include:

- **User authentication** for saving progress
- **Enhanced AI responses** for more immersive storytelling

---

## 🤝 Collaborator Guide

Thank you for your interest in contributing to the **AI-powered text adventure game** project! Below are the guidelines for contributing to both the **UI** and **API** repositories. Please follow these instructions to get started.

### **UI Repository: [GitHub Link to UI Repo](https://github.com/pratikIT95/text-game-llm-ui)**

The **UI repository** is responsible for the frontend of the text adventure game, built using **Angular** and **Angular Material**.

#### **Getting Started**

1. **Clone the Repository:**
   ```bash
   git clone https://github.com/pratikIT95/text-game-llm-ui.git
   cd text-game-llm-ui
   ```

2. **Install Dependencies:**
   Make sure you have **Node.js** and **npm** installed. Then, run:
   ```bash
   npm install
   ```

3. **Start the Development Server:**
   To run the UI locally, use:
   ```bash
   ng serve
   ```

4. **Open the Application:**
   Navigate to `http://localhost:4200` in your browser to view the application in action.

#### **How to Contribute**

- **Bug Fixes:** If you find any issues with the UI or functionality, feel free to open an issue or submit a pull request with your fix.
- **UI Enhancements:** If you have design suggestions or want to add new Angular Material components, please fork the repo, create a new branch, and submit your changes.
- **Testing:** Ensure that all components are working as expected by running:
  ```bash
  ng test
  ```

---

### **API Repository: [GitHub Link to API Repo](https://github.com/pratikIT95/text-game-llm-api)**

The **API repository** handles the backend logic, including **Spring Boot** for the API endpoints and integration with **Google Gemini** for AI-driven storytelling.

#### **Getting Started**

1. **Clone the Repository:**
   ```bash
   git clone https://github.com/pratikIT95/text-game-llm-api.git
   cd text-game-llm-api
   ```

2. **Install Dependencies:**
   Make sure you have **Java** and **Gradle** installed. Then, run:
   ```bash
   ./gradlew build
   ```

3. **Set Up the Google Gemini API Key:**
   Follow the steps in the guide above to obtain your **Gemini API Key** and add it to the `application-local.properties` file:
   ```properties
   gemini.api.key=YOUR_API_KEY_HERE
   ```

4. **Run the Backend:**
   Start the backend using the following command:
   ```bash
   ./gradlew bootRun
   ```

5. **Test the API Locally:**
   You can use tools like **Postman** or **curl** to test the various endpoints exposed by the API.

#### **How to Contribute**

- **Bug Fixes:** If you identify any bugs or issues with the API endpoints, create a pull request with a fix.
- **Enhancements:** If you have ideas for improving conversation management, the AI's responses, or the API's overall functionality, feel free to submit your changes after testing.
- **Integration with Other Services:** You are welcome to explore adding new AI features or integrating additional AI models to enhance the story generation.

---

### **General Contribution Guidelines**

- **Fork the Repositories:** To contribute, fork the repositories and create a branch for your changes.
- **Pull Requests:** Submit a pull request with a clear description of the changes you’ve made and the reasoning behind them.
- **Code Style:** Follow the existing coding styles and conventions in both repositories. Ensure your changes are well-documented and cleanly formatted.

We appreciate all contributions and encourage you to get involved!

---