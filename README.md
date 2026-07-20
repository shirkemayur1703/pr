import React, { useContext, useSyncExternalStore, useState, useEffect } from "react";

/* ---------------------------- MINI REDUX ENGINE -----------------------------
   Same shim as before, trimmed down. In your real project this is just
   `import { configureStore, createSlice } from "@reduxjs/toolkit";` +
   `import { Provider, useSelector } from "react-redux";` — nothing below
   this block changes. */
function createSlice({ name, initialState, reducers }) {
  const actions = {};
  const reducersByType = {};
  Object.keys(reducers).forEach((key) => {
    const type = `${name}/${key}`;
    actions[key] = (payload) => ({ type, payload });
    reducersByType[type] = reducers[key];
  });
  function reducer(state = initialState, action) {
    const fn = reducersByType[action.type];
    return fn ? fn(state, action) : state;
  }
  return { actions, reducer };
}

function configureStore({ reducer: reducersMap }) {
  let state = {};
  Object.keys(reducersMap).forEach((key) => {
    state = { ...state, [key]: reducersMap[key](undefined, { type: "@@INIT" }) };
  });
  const listeners = new Set();
  return {
    getState: () => state,
    dispatch: (action) => {
      const next = {};
      Object.keys(reducersMap).forEach((key) => {
        next[key] = reducersMap[key](state[key], action);
      });
      state = next;
      listeners.forEach((l) => l());
      return action;
    },
    subscribe: (listener) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
  };
}

const StoreContext = React.createContext(null);
function Provider({ store, children }) {
  return <StoreContext.Provider value={store}>{children}</StoreContext.Provider>;
}
function useSelector(selector) {
  const store = useContext(StoreContext);
  return useSyncExternalStore(store.subscribe, () => selector(store.getState()));
}
/* -------------------------- END MINI REDUX ENGINE --------------------------- */

/* FILE: uiSlice.js — the one piece of state this POC is about */
const uiSlice = createSlice({
  name: "ui",
  initialState: { showBanner: false },
  reducers: {
    toggleBanner: (state) => ({ ...state, showBanner: !state.showBanner }),
  },
});
const { toggleBanner } = uiSlice.actions;

/* FILE: store.js */
const store = configureStore({ reducer: { ui: uiSlice.reducer } });

/* FILE: ThirdPartyChatbot.jsx — owned by another team. It just renders
   whatever you hand it via `customSlot`. It has no idea what's inside. */
function ThirdPartyChatbot({ customSlot }) {
  return (
    <div style={{ border: "1px solid #2a2a2a", borderRadius: 10, padding: 14, background: "#111" }}>
      <div style={{ fontSize: 12, color: "#777", marginBottom: 10 }}>
        third-party child — renders whatever "customSlot" prop it's given
      </div>
      <div>{customSlot}</div>
      {!customSlot && <div style={{ fontSize: 12, color: "#444" }}>(nothing in the slot right now)</div>}
    </div>
  );
}

/* A real component, to prove this is a genuine mount/unmount — not just a
   CSS show/hide. The timestamp only appears once per mount; toggling off
   and back on gets a NEW timestamp because it's a fresh mount. */
function EscalationBanner() {
  const [mountedAt] = useState(() => new Date().toLocaleTimeString());
  useEffect(() => {
    console.log("EscalationBanner mounted");
    return () => console.log("EscalationBanner unmounted");
  }, []);
  return (
    <div style={{ background: "#7f1d1d", color: "#fecaca", padding: "10px 12px", borderRadius: 6, fontSize: 13 }}>
      🚨 Escalation banner — mounted at {mountedAt}
    </div>
  );
}

/* FILE: ChatbotWrapper.jsx */
function ChatbotWrapperInner() {
  const showBanner = useSelector((s) => s.ui.showBanner);

  return (
    <div style={{ fontFamily: "ui-sans-serif, system-ui, sans-serif", background: "#000", minHeight: "100vh", padding: 24, color: "#eee" }}>
      <div style={{ maxWidth: 420, margin: "0 auto" }}>
        <button
          onClick={() => store.dispatch(toggleBanner())}
          style={{
            background: "transparent",
            border: `1px solid ${showBanner ? "#f87171" : "#4ade80"}55`,
            color: showBanner ? "#f87171" : "#4ade80",
            borderRadius: 6,
            padding: "8px 12px",
            fontSize: 13,
            cursor: "pointer",
            marginBottom: 16,
          }}
        >
          store.dispatch(toggleBanner()) → showBanner: {String(showBanner)}
        </button>

        <ThirdPartyChatbot customSlot={showBanner ? <EscalationBanner /> : null} />
      </div>
    </div>
  );
}

export default function App() {
  return (
    <Provider store={store}>
      <ChatbotWrapperInner />
    </Provider>
  );
}
