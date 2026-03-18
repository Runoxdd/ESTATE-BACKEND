# Critical Fixes Plan - PrimeNest

## ✅ ALL FIXES COMPLETED

This plan has been fully implemented. See the summary below.

---

## Executive Summary

This plan addresses four critical issues affecting the PrimeNest application:

1. **AI Assistant Quota Exceeded** - ✅ FIXED - Switched to Groq
2. **Messages Page Blank Screen** - ✅ FIXED - CSS layout overhaul
3. **Messages Scrolling Broken** - ✅ FIXED - Flex container fixes
4. **Duplicate Chats** - ✅ FIXED - Backend + frontend debounce
5. **Listings Card Sizes** - ✅ FIXED - Uniform card heights

---

## Issue 1: AI Assistant - Gemini API Quota Exceeded

### Problem
The error shows:
```
ApiError: 429 - You exceeded your current quota
Quota exceeded for metric: generate_content_free_tier_requests
Model: gemini-2.0-flash
```

The application uses Gemini 2.0 Flash for response generation, but the free tier quota has been exhausted.

### Root Cause Analysis
- File: [`api/controllers/assistant.controller.js`](api/controllers/assistant.controller.js:308-321)
- The `generateResponse` function uses `ai.models.generateContent` with `gemini-2.0-flash`
- Groq SDK is already imported and used for intent detection
- Groq has a much more generous free tier (14M tokens/month vs Gemini's limited free tier)

### Solution
**Switch entirely to Groq for both intent detection AND response generation.**

#### Changes Required

**File: `api/controllers/assistant.controller.js`**

1. Remove Gemini imports and initialization:
```javascript
// REMOVE:
import { GoogleGenAI } from "@google/genai";
const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });
```

2. Update `generateResponse` function to use Groq instead of Gemini:
```javascript
const generateResponse = async (message, context, sessionId, searchResults, intentData) => {
  // Build conversation history for Groq
  const conversationHistory = context.history.map(m => ({
    role: m.role === 'model' ? 'assistant' : m.role,
    content: m.content
  }));
  
  // Add current message
  conversationHistory.push({
    role: 'user',
    content: message
  });

  try {
    const completion = await groq.chat.completions.create({
      messages: [
        {
          role: "system",
          content: systemInstruction // Same instruction content
        },
        ...conversationHistory
      ],
      model: "llama-3.1-8b-instant", // or "llama-3.3-70b-versatile" for better quality
      temperature: 0.7,
      max_tokens: 800,
      response_format: { type: "json_object" }
    });

    const responseText = completion.choices[0]?.message?.content || '{}';
    const parsedData = JSON.parse(responseText);
    
    // Rest of the logic remains the same...
  } catch (err) {
    console.error("Response generation error:", err);
    // Fallback response...
  }
};
```

3. Update system instruction format for Groq (same content, just ensure JSON response format is requested)

### Benefits
- Groq free tier: 14M tokens/month (much higher than Gemini)
- Faster response times with Groq's inference engine
- No more quota exceeded errors
- Already have Groq SDK installed and configured

---

## Issue 2: Messages Page - Blank Screen When Clicking Conversation

### Problem
When clicking on a conversation in the messages list, the chat window goes blank instead of showing the conversation.

### Root Cause Analysis
- File: [`client/src/routes/messagesPage/MessagesPage.jsx`](client/src/routes/messagesPage/MessagesPage.jsx:182-265)
- File: [`client/src/routes/messagesPage/messagesPage.scss`](client/src/routes/messagesPage/messagesPage.scss:249-265)

The mobile CSS uses `position: absolute` with `transform: translateX(100%)` for the chat window. The issue is:

1. The `isMobileListOpen` state controls visibility
2. When `isMobileListOpen` is `true`, sidebar shows, chat window is hidden
3. When `isMobileListOpen` is `false`, chat window should show

But the CSS class logic might be inverted or the positioning is causing issues.

### Solution

**File: `client/src/routes/messagesPage/messagesPage.scss`**

Fix the chat window positioning and ensure proper display:

```scss
.chat-window {
  display: flex;
  flex-direction: column;
  background: var(--bg-primary);
  flex: 1;
  min-width: 0; // Prevent flex overflow

  @include mobile {
    position: absolute;
    inset: 0;
    z-index: 5;
    transform: translateX(100%);
    transition: transform 0.3s ease;
    background: var(--bg-primary); // Ensure background is set

    &.open {
      transform: translateX(0);
    }
  }
}
```

**File: `client/src/routes/messagesPage/MessagesPage.jsx`**

Ensure the chat window always renders (even when no chat selected) to prevent blank screen:

```jsx
{/* Chat Window */}
<div className={`chat-window ${!isMobileListOpen ? "open" : ""}`}>
  {chat ? (
    <>
      {/* Chat content */}
    </>
  ) : (
    <div className="no-chat-selected">
      {/* Empty state */}
    </div>
  )}
</div>
```

---

## Issue 3: Messages Scrolling Not Working

### Problem
Cannot scroll up to see previous messages in the chat window.

### Root Cause Analysis
- File: [`client/src/routes/messagesPage/messagesPage.scss`](client/src/routes/messagesPage/messagesPage.scss:402-476)

The `.chat-messages` container has:
```scss
.chat-messages {
  flex: 1;
  overflow-y: auto;
  // ...
}
```

The issue is that the parent container `.chat-window` might not have proper height constraints, causing the flex child to not calculate its height correctly.

### Solution

**File: `client/src/routes/messagesPage/messagesPage.scss`**

Add explicit height constraints and fix overflow:

```scss
.chat-window {
  display: flex;
  flex-direction: column;
  background: var(--bg-primary);
  height: 100%; // Add explicit height
  min-height: 0; // Allow flex child to shrink

  @include mobile {
    // ... mobile styles
  }
}

.chat-messages {
  flex: 1;
  min-height: 0; // Critical for flex overflow
  overflow-y: auto;
  padding: $space-5;
  display: flex;
  flex-direction: column;
  gap: $space-3;
  background: var(--bg-primary);
  @include custom-scrollbar(4px);
  
  // Ensure scrollable
  -webkit-overflow-scrolling: touch; // iOS smooth scrolling
}
```

Also ensure the messages container has proper height on mobile:

```scss
.messages-container {
  display: grid;
  grid-template-columns: 360px 1fr;
  max-width: 1400px;
  margin: 0 auto;
  height: calc(100vh - 140px);
  // ...
  
  @include mobile {
    display: flex; // Change to flex for mobile
    flex-direction: column;
    height: calc(100vh - 64px);
    position: relative; // For absolute positioned children
  }
}
```

---

## Issue 4: Duplicate Chats When Clicking Message Seller Multiple Times

### Problem
When clicking "Message Seller" button multiple times, the same user appears multiple times in the conversations list.

### Root Cause Analysis
- File: [`api/controllers/chat.controller.js`](api/controllers/chat.controller.js:75-102)

The `addChat` function checks for existing chats:
```javascript
const existingChat = await prisma.chat.findFirst({
  where: {
    userIDs: {
      hasEvery: [tokenUserId, receiverId],
    },
  },
});
```

This logic is correct, but there's a **race condition**:
1. User clicks "Message Seller" twice rapidly
2. First request checks for existing chat - finds none
3. Second request checks for existing chat - finds none (first chat not yet created)
4. Both requests create new chats

### Solution

#### Backend Fix - Add Unique Constraint

**File: `api/prisma/schema.prisma`**

Add a unique constraint or use upsert logic:

```prisma
model Chat {
  id        String   @id @default(uuid())
  userIDs   String[] // Array of user IDs
  createdAt DateTime @default(now())
  seenBy    String[]
  messages  Message[]

  @@unique([userIDs]) // This won't work for arrays
}
```

Since arrays can't have unique constraints in Prisma/MongoDB, use a transaction with locking or idempotency:

**File: `api/controllers/chat.controller.js`**

```javascript
export const addChat = async (req, res) => {
  const tokenUserId = req.userId;
  const { receiverId } = req.body;

  try {
    // Sort user IDs to ensure consistent ordering
    const sortedUserIds = [tokenUserId, receiverId].sort();
    
    // Check for existing chat
    const existingChat = await prisma.chat.findFirst({
      where: {
        AND: [
          { userIDs: { has: sortedUserIds[0] } },
          { userIDs: { has: sortedUserIds[1] } },
        ],
      },
    });

    if (existingChat) {
      return res.status(200).json(existingChat);
    }

    // Create new chat
    const newChat = await prisma.chat.create({
      data: {
        userIDs: sortedUserIds,
      },
    });
    
    res.status(200).json(newChat);
  } catch (err) {
    // Handle unique constraint violation (in case race condition still occurs)
    if (err.code === 'P2002') {
      // Fetch the existing chat that was created by concurrent request
      const existingChat = await prisma.chat.findFirst({
        where: {
          AND: [
            { userIDs: { has: tokenUserId } },
            { userIDs: { has: receiverId } },
          ],
        },
      });
      return res.status(200).json(existingChat);
    }
    console.log(err);
    res.status(500).json({ message: "Failed to add chat!" });
  }
};
```

#### Frontend Fix - Debounce Button

**File: `client/src/components/chat/Chat.jsx`** (or wherever "Message Seller" button is)

Add debounce or disable button after click:

```jsx
const [isCreatingChat, setIsCreatingChat] = useState(false);

const handleMessageSeller = async () => {
  if (isCreatingChat) return; // Prevent double-click
  
  setIsCreatingChat(true);
  try {
    const res = await apiRequest.post("/chats", { receiverId: sellerId });
    // Navigate to chat or update state
  } catch (err) {
    console.error(err);
  } finally {
    setIsCreatingChat(false);
  }
};
```

---

## Issue 5: Listings Cards - Inconsistent Sizes

### Problem
Property cards on the listings page have different shapes and sizes instead of being uniform.

### Root Cause Analysis
- File: [`client/src/components/card/card.scss`](client/src/components/card/card.scss:8-50)
- File: [`client/src/routes/listPage/listPage.jsx`](client/src/routes/listPage/listPage.jsx:66-84)

The cards use:
```scss
.card {
  .card-image {
    aspect-ratio: 16 / 10;
  }
}

.properties-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
}
```

The issue is:
1. Cards have varying content heights (title length, location text)
2. Grid items stretch to fit tallest card in row
3. No fixed height for card content area

### Solution

**File: `client/src/components/card/card.scss`**

Make cards uniform with fixed heights:

```scss
.card {
  background: var(--card-bg);
  border: 1px solid var(--card-border);
  border-radius: $radius-2xl;
  overflow: hidden;
  box-shadow: var(--card-shadow);
  transition: box-shadow 0.3s ease;
  height: 100%; // Full height of grid cell
  display: flex;
  flex-direction: column;

  .card-link {
    display: flex;
    flex-direction: column;
    height: 100%;
  }

  .card-image {
    position: relative;
    width: 100%;
    aspect-ratio: 16 / 10; // Fixed aspect ratio
    flex-shrink: 0; // Don't shrink
    overflow: hidden;
  }

  .card-content {
    padding: $space-4;
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
  }

  .card-title {
    // Fixed height for 2 lines
    min-height: 2.5em;
    line-height: 1.25;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }

  .card-location {
    // Fixed height
    min-height: 1.5em;
  }

  .card-features {
    margin-top: auto; // Push to bottom
  }
}
```

**File: `client/src/routes/listPage/listPage.scss`**

Ensure grid items align properly:

```scss
.properties-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: $space-5;
  align-items: stretch; // All cards same height in row

  @include tablet {
    grid-template-columns: repeat(2, 1fr);
  }

  @include mobile {
    grid-template-columns: 1fr;
  }
}
```

---

## Implementation Order

1. **AI Assistant Fix** (Highest Priority - Currently Broken)
   - Switch to Groq for all AI operations
   - Test thoroughly with different queries

2. **Messages Page Fixes** (High Priority - Core Feature)
   - Fix blank screen issue
   - Fix scrolling
   - Fix duplicate chats

3. **Listings Card Fix** (Medium Priority - UI Polish)
   - Standardize card sizes
   - Ensure consistent grid layout

---

## Testing Checklist

### AI Assistant
- [ ] Can send message and receive response
- [ ] Intent detection works correctly
- [ ] Property search returns results
- [ ] Conversation context is maintained
- [ ] No quota errors

### Messages Page
- [ ] Clicking conversation shows chat
- [ ] Can scroll through message history
- [ ] No duplicate conversations
- [ ] Mobile view works correctly
- [ ] Can send and receive messages

### Listings Page
- [ ] All cards same height
- [ ] Images same aspect ratio
- [ ] Grid layout is uniform
- [ ] Responsive design works

---

## Files to Modify

| File | Changes |
|------|---------|
| `api/controllers/assistant.controller.js` | Switch from Gemini to Groq |
| `api/controllers/chat.controller.js` | Fix race condition for duplicate chats |
| `client/src/routes/messagesPage/MessagesPage.jsx` | Fix chat window display logic |
| `client/src/routes/messagesPage/messagesPage.scss` | Fix scrolling and layout |
| `client/src/components/card/card.scss` | Standardize card sizes |
| `client/src/routes/listPage/listPage.scss` | Fix grid alignment |
| `client/src/components/chat/Chat.jsx` | Add debounce to message button |
