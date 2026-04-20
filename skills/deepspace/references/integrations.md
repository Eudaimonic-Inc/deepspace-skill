# DeepSpace Integrations Reference

All integrations are called through the api-worker proxy:

```typescript
import { integration } from 'deepspace'
const result = await integration.post('<integration-name>/<endpoint-name>', { ...params })
// Returns: { success: true, data: ... } or { success: false, error: "..." }
```

**Endpoint keys are two segments: `<integration>/<endpoint>`.** Use the exact names below — do not invent or paraphrase.

## AI / LLM

### anthropic
- `anthropic/chat-completion` — Claude chat completions

### openai
- `openai/chat-completion` — GPT chat completions
- `openai/generate-image` — DALL-E image generation

### gemini
- `gemini/generate-image` — Gemini image generation

## Search / Web

### exa
- `exa/search` — web search
- `exa/answer` — direct Q&A
- `exa/contents` — fetch page contents
- `exa/news-search` — news search
- `exa/research` — research agent

### firecrawl
- `firecrawl/scrape` — scrape a URL
- `firecrawl/crawl` — crawl a site
- `firecrawl/map` — site map
- `firecrawl/search` — web search

### serpapi
- `serpapi/search` — Google search
- `serpapi/web-search` — generic web search
- `serpapi/events` — events search
- `serpapi/flights` — flights search
- `serpapi/hotels` — hotels search
- `serpapi/places-search` — places search
- `serpapi/places-reviews` — place reviews
- `scholar/search-papers` — Google Scholar papers
- `scholar/search-authors` — Google Scholar authors
- `scholar/get-author-details`
- `scholar/get-author-papers`
- `scholar/get-citation-details`

### websearch
- `websearch/advanced-search` — advanced web search

### wikipedia
- `wikipedia/search-pages`
- `wikipedia/get-page-summary`
- `wikipedia/get-page-content`
- `wikipedia/get-random-page`

## Weather / Location

### openweathermap
- `openweathermap/current` — current weather
- `openweathermap/forecast` — forecast
- `openweathermap/geocoding` — city → lat/lng

## News

### newsapi
- `newsapi/top-headlines`
- `newsapi/search-everything`

## Media — Images / Video

### freepik
- `freepik/generate-image-flux-dev`
- `freepik/generate-image-flux-pro`
- `freepik/generate-image-flux-2-pro`
- `freepik/generate-image-flux-2-turbo`
- `freepik/generate-image-hyperflux`
- `freepik/generate-image-mystic`
- `freepik/generate-image-runway`
- `freepik/generate-image-seedream`
- `freepik/generate-image-seedream-v4`
- `freepik/generate-image-seedream-v4-5`
- `freepik/generate-image-z-image`
- `freepik/generate-video`
- `freepik/text-to-image-classic`
- `freepik/image-expand`
- `freepik/image-relight`
- `freepik/image-style-transfer`
- `freepik/upscale-image-precision`
- `freepik/remove-background`
- `freepik/download-icons`
- `freepik/download-stock-images`
- `freepik/download-stock-videos`

### submagic
- `submagic/create-video`
- `submagic/get-project`
- `submagic/wait-for-completion`

### cloudconvert
- `cloudconvert/convert-file`

## Audio / Speech

### elevenlabs
- `elevenlabs/generate-speech`
- `elevenlabs/list-voices`
- `elevenlabs/create-agent`
- `elevenlabs/get-signed-url`

### speech
- `speech/text-to-speech`
- `speech/speech-to-text`

## Communication

### email
- `email/send` — send transactional email

### slack
Pass a workspace bot/user `accessToken` in the request body — there is no OAuth callback flow from the SDK side; obtain the token out of band.
- `slack/send-message`
- `slack/list-channels`
- `slack/channel-history`
- `slack/team-info`

### livekit (real-time audio/video rooms)
- `livekit/create-room`
- `livekit/delete-room`
- `livekit/list-rooms`
- `livekit/generate-token`

## Google (OAuth required)

### gmail
- `google/gmail-list`
- `google/gmail-get`
- `google/gmail-search`
- `google/gmail-send`

### drive
- `google/drive-list`
- `google/drive-get`

### calendar
- `google/calendar-list-events`
- `google/calendar-create-event`
- `google/calendar-delete-event`

### contacts
- `google/contacts-list`

## Social

### github
- `github/get-user`
- `github/get-public-user`
- `github/get-user-repos`
- `github/get-user-public-repos`
- `github/get-repository`
- `github/get-repository-contents`
- `github/get-repository-readme`
- `github/get-repository-tree`
- `github/get-repository-commits`
- `github/get-repository-contributors`
- `github/get-repository-languages`
- `github/get-repository-issues`
- `github/get-repository-pulls`
- `github/get-pull-request`
- `github/get-pull-request-files`
- `github/get-pull-request-reviews`
- `github/get-commit`
- `github/search-repositories`

### linkedin
- `linkedin/search-profiles`
- `linkedin/analyze-profile-url`

### instagram
- `instagram/extract-content`

### tiktok
- `tiktok/post-video`
- `tiktok/cancel-scheduled-post`
- `tiktok/get-scheduled-posts`

### youtube
- `youtube/search-videos`
- `youtube/get-video-details`
- `youtube/get-trending-videos`

## Finance

### finance
- `finnhub/stock-price`
- `alphavantage/search-symbols`
- `coinbase/crypto-price`
- `coinbase/search-crypto`
- `coinbase/search-currencies`

### polymarket
- `polymarket/markets`
- `polymarket/market-detail`
- `polymarket/events`
- `polymarket/event-detail`
- `polymarket/search`
- `polymarket/tags`
- `polymarket/comments`
- `polymarket/orderbook`
- `polymarket/price`
- `polymarket/prices`
- `polymarket/price-history`
- `polymarket/trades`

## Sports

### american football
- `api-american-football/games`
- `api-american-football/games-events`
- `api-american-football/games-statistics-players`
- `api-american-football/games-statistics-teams`
- `api-american-football/leagues`
- `api-american-football/teams`
- `api-american-football/players`
- `api-american-football/players-statistics`
- `api-american-football/standings`
- `api-american-football/standings-conferences`
- `api-american-football/standings-divisions`
- `api-american-football/injuries`
- `api-american-football/odds`
- `api-american-football/odds-bookmakers`

### football (soccer)
- `api-football/fixtures`
- `api-football/fixtures-events`
- `api-football/fixtures-lineups`
- `api-football/fixtures-statistics`
- `api-football/fixtures-headtohead`
- `api-football/leagues`
- `api-football/teams`
- `api-football/teams-statistics`
- `api-football/players`
- `api-football/players-squads`
- `api-football/players-topscorers`
- `api-football/players-topassists`
- `api-football/standings`
- `api-football/predictions`
- `api-football/injuries`
- `api-football/transfers`
- `api-football/coachs`
- `api-football/venues`
- `api-football/countries`

### basketball
- `api-basketball/games`
- `api-basketball/games-h2h`
- `api-basketball/games-statistics-players`
- `api-basketball/games-statistics-teams`
- `api-basketball/leagues`
- `api-basketball/teams`
- `api-basketball/players`
- `api-basketball/standings`
- `api-basketball/standings-groups`
- `api-basketball/standings-stages`
- `api-basketball/statistics`
- `api-basketball/odds`
- `api-basketball/bookmakers`
- `api-basketball/countries`

### baseball
- `api-baseball/games`
- `api-baseball/games-h2h`
- `api-baseball/leagues`
- `api-baseball/teams`
- `api-baseball/teams-statistics`
- `api-baseball/standings`
- `api-baseball/standings-groups`
- `api-baseball/standings-stages`
- `api-baseball/odds`
- `api-baseball/odds-bookmakers`
- `api-baseball/countries`

### f1
- `f1/season-schedule`
- `f1/race-weekend`
- `f1/latest-race`
- `f1/race-results`
- `f1/qualifying`
- `f1/sprint`
- `f1/lap-times`
- `f1/pit-stops`
- `f1/driver-standings`
- `f1/constructor-standings`
- `f1/all-drivers`
- `f1/all-constructors`
- `f1/circuit-info`

## Shopping

### amazon
- `amazon/search-products`

## Transit

### mta (NYC MTA)
- `mta/feed`
- `mta/list-feeds`
- `mta/alerts`
- `mta/arrivals`

## Science / Space

### nasa
- `nasa/apod` — astronomy picture of the day
- `nasa/neo-feed` — near-earth objects feed
- `nasa/neo-lookup`
- `nasa/neo-browse`
- `nasa/cme` — coronal mass ejections
- `nasa/flr` — solar flares
- `nasa/gst` — geomagnetic storms

## Documents

### latex
- `latex-compiler/compile` — compile LaTeX to PDF

## Response Format

All endpoints return:
```typescript
{ success: true, data: <endpoint-specific> } | { success: false, error: string }
```

`data` shape varies by endpoint. Common pattern: `data` is the raw upstream response (often an array for list endpoints, object for detail endpoints). Do not assume nested keys like `data.list` or `data.results` without verifying — check `Array.isArray(result.data)` first.

## OAuth

Currently only `google` requires OAuth. Users connect once; tokens are managed by the platform. If a Google endpoint returns `{ success: false, error: 'not_connected' }`, prompt the user to connect their Google account.
