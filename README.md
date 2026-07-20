import { useState, useEffect } from "react";
import { createSlice, configureStore } from "@reduxjs/toolkit";
import { Provider, useSelector, useDispatch } from "react-redux";

/* ---------------------------- uiSlice ---------------------------- */
const uiSlice = createSlice({
  name: "ui",
  initialState: { showBanner: false },
  reducers: {
    toggleBanner(state) {
      state.showBanner = !state.showBanner; // Immer handles the immutability
    },
    setBanner(state, action) {
      state.showBanner = action.payload;
    },
  },
});
const { toggleBanner, setBanner } = uiSlice.actions;

/* ---------------------------- store ---------------------------- */
const store = configureStore({
  reducer: {
    ui: uiSlice.reducer,
  },
});

/* ------------------------ ThirdPartyChatbot ------------------------
   Stand-in for the component you don't control. It just renders
   whatever you hand it via "customSlot" — it has no idea what's inside. */
function ThirdPartyChatbot({ customSlot }) {
  return (
    <div style={{ border: "1px solid #2a2a2a", borderRadius: 10, padding: 14 }}>
      <div style={{ fontSize: 12, color: "#777", marginBottom: 10 }}>
        third-party child — renders whatever "customSlot" prop it's given
      </div>
      <div>{customSlot}</div>
    </div>
  );
}

/* ------------------------ EscalationBanner ------------------------
   The timestamp is set once, on mount. If toggling off and back on
   shows a NEW timestamp, that proves this is a real unmount/remount —
   not a hidden element being revealed. */
function EscalationBanner() {
  const [mountedAt] = useState(() => new Date().toLocaleTimeString());

  useEffect(() => {
    console.log("EscalationBanner mounted");
    return () => console.log("EscalationBanner unmounted");
  }, []);

  return (
    <div style={{ background: "#7f1d1d", color: "#fecaca", padding: "10px 12px", borderRadius: 6 }}>
      🚨 Escalation banner — mounted at {mountedAt}
    </div>
  );
}

/* ---------------------------- ChatbotWrapper ---------------------------- */
function ChatbotWrapper() {
  const showBanner = useSelector((state) => state.ui.showBanner);
  const dispatch = useDispatch();

  return (
    <div style={{ maxWidth: 420, margin: "40px auto", fontFamily: "sans-serif" }}>
      <button onClick={() => dispatch(toggleBanner())}>
        showBanner: {String(showBanner)} (click to toggle)
      </button>

      <div style={{ marginTop: 16 }}>
        <ThirdPartyChatbot customSlot={showBanner ? <EscalationBanner /> : null} />
      </div>
    </div>
  );
}

/* ---------------------------- App (default export) ---------------------------- */
export default function App() {
  return (
    <Provider store={store}>
      <ChatbotWrapper />
    </Provider>
  );
}
