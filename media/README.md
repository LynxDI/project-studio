# Media

Public-facing images for the Project Studio README and the VS Code Marketplace listing.

The README references images by **absolute** URL
(`https://raw.githubusercontent.com/LynxDI/project-studio/main/media/<name>.png`) — relative
paths render on GitHub but come up blank on the Marketplace, so always use the full raw URL.

## Current screenshots

| File | What it shows |
|------|---------------|
| `dashboard.png` | The **Home dashboard** — stat tiles (active projects, events, commits, AI sessions, known cost, provider health), the **AI Fleet** card with per-session run states, **Usage & Cost** with its calc-type labels, and **Provider Activity** comparing providers with `N/A` where a provider has no data. |
| `fleet.png` | The **AI Fleet** tree in the activity bar, grouped Active / Waiting / Recent / Failed — "needs you" (waiting on a human) is shown distinctly from waiting on a tool. |
| `workspace.png` | The whole VS Code window: the Project Studio container with the Projects, Timeline, AI Fleet, and Unassigned trees, plus the timeline filter picker. |

All images use **synthetic fixture data** — the "E2E Fixture Project", invented session titles, and
a scripted throwaway Git repository. No real user data, no PII, and no real customer project ever
appears in a published image.

## How they're captured

The images are produced by the same end-to-end suite that verifies the UI, so a published
screenshot is provably a state the tests actually exercised — not a mock-up.

```bash
# from the private source repo
npm run shots:ui                    # real VS Code (1.130) driven over CDP by the e2e-ui suite
                                    #   -> tools/ui-shots/e2e-*.png
node tools/crop-marketing.mjs       # crop editor furniture -> public/media/*.png
```

`npm run shots:ui` builds the bundles, opens a temporary fixture workspace containing a real Git
repository, runs the actual registration and import pipeline, seeds synthetic agent sessions, and
screenshots the workbench over the Chrome DevTools Protocol. The crop step removes the chat side
bar and any notification toast — editor furniture, not product — while leaving the captured pixels
otherwise untouched.

A second harness, `npm run shots`, renders the dashboard webview headlessly (no desktop session
required) for fast design iteration:

```bash
npm run shots                       # -> tools/ui-shots/home-{empty,ai-pending,populated}.png
```

> When updating these images, re-run the capture rather than editing pixels: the point of shipping
> test-derived screenshots is that the listing cannot drift from the product.
