---
name: react-native-expo-patterns
description: Best practices for the mobile app in shopoo (React Native + Expo + TypeScript). Use when building or reviewing the mobile/ repo — covers typed navigation, TanStack Query + Zustand state, secure token storage + axios refresh interceptor, FlatList performance, image picking/upload, Socket.io-client lifecycle tied to auth/appState, push notifications (FCM via expo-notifications), and env config. Pair with marketplace-conventions.
---

# React Native (Expo) Patterns

> `mobile/` is the consumer app: browse/search, post listings (with photos), real-time chat, push. Optimise for I/O, smooth lists, and resilient connectivity.

## Foundations
- **Expo** (managed; go bare only if a native module like FCM demands it). **TypeScript** throughout.
- **Navigation:** React Navigation with typed param lists. Structure: `AuthStack` (logged-out) vs `MainTabs` (Home, Search, Post, Chat, Profile); detail screens in nested stacks.

## State & data
- **Server state → TanStack Query** (caching, retry, `useInfiniteQuery` for listing feeds). **Global client state → Zustand** (auth user, socket status, unread counts). Don't duplicate server data into Zustand.
- One **axios instance** with an interceptor that attaches the access token and, on 401, refreshes once (queueing concurrent requests) then retries; on refresh failure, logs out.

## Auth & storage
- Access token in memory/Zustand; **refresh token in `expo-secure-store`** (never AsyncStorage for secrets). Clear both on logout.

## Lists & images
- `FlatList`/`FlashList` with stable `keyExtractor`, `getItemLayout` where possible, `windowSize`/`initialNumToRender` tuned, memoized row components. Paginate via `onEndReached` + `useInfiniteQuery`.
- Images: `expo-image` (caching) for display; `expo-image-picker` to select; upload multipart to Media Service with progress; show optimistic placeholders.

## Realtime (chat)
- `socket.io-client` connected **after** login with the JWT in `auth`; reconnect with backoff. Tie lifecycle to `AppState` (disconnect/quiet in background, resync on foreground). Reconcile live `message:new` with React Query message cache; send `clientMsgId` for idempotency; render optimistic sent messages.

## Push notifications
- `expo-notifications`; request permission, get the device/FCM token, register via `POST /devices`. Handle foreground + tapped notifications → deep link to the right screen (ChatRoom / ListingDetail). Re-register token on change.

## Performance & env
- Keep heavy work off the JS thread; `useMemo`/`useCallback` for expensive renders and stable props. Avoid anonymous inline functions in hot list rows.
- Config via `app.config.ts` + `expo-constants` (`extra`); separate dev/prod API base URLs. `NEXT_PUBLIC`-style: nothing secret ships in the bundle.

## Don't
Don't store secrets in AsyncStorage · don't refetch with Query and also cache the same data in Zustand · don't keep the socket open in the background indefinitely · don't render unbounded lists without virtualization · don't block the JS thread with sync work.
