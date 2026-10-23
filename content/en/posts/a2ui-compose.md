---
title: "A2UI Hands-On: the Agent Draws the UI, Jetpack Compose Renders It"
date: 2026-09-30T09:15:53+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin", "AI Agent"]
summary: "Same JSON, same app, same run."
cover:
    image: "/images/a2ui-compose/cover.png"
    alt: "A2UI Hands-On: the Agent Draws the UI, Jetpack Compose Renders It"
    relative: false
---

## Two screenshots first

Same JSON, same app, same run.

The first: the form is full. Guest Ada Lovelace, date Oct 1, 2026, time 7:30 PM, party of 2, two buttons underneath reading Cancel and Confirm booking. Native Material 3 controls all the way down.

![Fields populated correctly](/images/a2ui-compose/fields-filled.png)

The second: the same form, every field empty. Not just empty. The labels on both date-time controls are gone too, leaving two gray rounded boxes.

![Same JSON, every field empty](/images/a2ui-compose/fields-empty.png)

The only difference is when I sent the `updateDataModel` message. Send it in the same batch as the components and the fields get dropped. Send it a second later, on its own, and the values land. I will come back to this. It is the most valuable hole I fell into this time.

## What A2UI is

One sentence, their words:

> A2UI (Agent to UI) is a declarative UI protocol for agent-driven interfaces. AI agents generate rich, interactive UIs that render natively across platforms (web, mobile, desktop) without executing arbitrary code.

The agent does not emit HTML or JavaScript. It emits declarative JSON describing what the interface looks like, and the client renders it with its own native components. The scenario they use to contrast it is the classic one:

> User: "Book a table for 2 tomorrow at 7pm"
> Agent: "Okay, for what day?"
> User: "Tomorrow"
> Agent: "What time?"

Three round trips to fill one form. Swap in an agent that generates the form itself, with a date picker, a time picker and a submit button, and the user is operating an interface instead of talking to one.

The smartest design in this protocol, I think, is the catalog. It does not let the agent invent UI freely. The client declares a contract first: these are the components, these are the properties, these are the local functions. The agent can only pick and compose inside that contract. The spec puts it plainly:

> Rather than having an agent generate arbitrary UI code or invent unregistered components, the catalog acts as a contract that defines the specific UI elements, properties, and functions available to the agent.

Add the sends-data-not-code property and the security math gets easy: no iframe, no arbitrary code execution.

## The official renderer on the Compose side

On September 23, AndroidX shipped an A2UI renderer for Jetpack Compose, 1.0.0-alpha01. I checked each of these artifacts on Google Maven; they all exist:

```kotlin
implementation("androidx.a2ui.compose:compose-runtime:1.0.0-alpha01")  // fine-grained recomposition on Compose Snapshot state
implementation("androidx.a2ui.compose:compose-ui:1.0.0-alpha01")       // components and catalog definitions
implementation("androidx.a2ui:a2ui-model:1.0.0-alpha01")               // protocol data layer
implementation("androidx.a2ui:a2ui-engine:1.0.0-alpha01")              // parsing, schema validation
implementation("androidx.compose.material3:material3-a2ui:1.0.0-alpha01")  // Material 3 implementation of the Basic Catalog
```

`material3-a2ui` is the finished product: the Material 3 implementation of the Basic Catalog, one line and it runs. What you have to supply is image, video and audio players, plus callbacks for opening URLs and formatting localized text. Leaving out Coil and Media3 is deliberate:

> The A2UI libraries intentionally don't bundle external media and networking dependencies, such as Coil, Glide, or Media3. Instead, you provide your own renderers.

One more thing KMP developers will ask about immediately. I unpacked the Gradle module metadata for `compose-runtime`:

```text
- releaseVariantReleaseApiPublication     type=aar
- releaseVariantReleaseRuntimePublication type=aar
```

Only aar. No klib, no JVM variant. This release is Android-only, so Compose Multiplatform cannot use it today.

## Minimal wiring

Two objects on the data side: a parser and a processor.

```kotlin
private val parser = A2uiMessageParser()

private val catalog = materialA2uiBasicCatalogV1(
    image = MaterialA2uiBasicCatalogV1Defaults.image { url, _, _, modifier, _ ->
        Placeholder("image", url, modifier)
    },
    urlOpener = { url -> Log.i(TAG, "open $url") },
    messageFormatter = { pattern, locale, args -> /* MessageFormat */ },
    localeProvider = A2uiLocaleProvider.Default,
)

private val processor = A2uiMessageProcessor(catalogs = listOf(catalog))
val surfaces = processor.activeSurfaces

init {
    viewModelScope.launch(Dispatchers.Default) { processor.collectMessages() }
    viewModelScope.launch(Dispatchers.Default) {
        processor.outboundEvents.collect { Log.e(TAG, "OUTBOUND: $it") }
    }
}
```

The UI side is one loop, handing each surface to `A2uiSurface`:

```kotlin
surfaces.forEach { surface ->
    key(surface.id) {
        A2uiSurface(surfaceModel = surface, modifier = Modifier.fillMaxWidth())
    }
}
```

What goes in is three protocol messages. `createSurface` opens the canvas, `updateComponents` describes the component tree, `updateDataModel` supplies the bound values:

```json
{ "version": "v0.9",
  "createSurface": { "surfaceId": "booking",
                     "catalogId": "https://a2ui.org/specification/v0_9/catalogs/basic/catalog.json" } }

{ "version": "v0.9",
  "updateComponents": { "surfaceId": "booking",
    "components": [
      { "id": "root", "component": "Column", "children": ["title", "nameField", "actions"] },
      { "id": "title", "component": "Text", "text": "Book a Table", "variant": "h1" },
      { "id": "nameField", "component": "TextField", "label": "Guest name",
        "value": { "path": "/booking/name" }, "variant": "shortText" }
    ] } }
```

Note that `value` on a component is not the value itself but a `path` into the data model. Which is exactly why whether that third message lands determines whether your form is full or empty.

## Once it runs, the pits come one after another

Every item below is something I actually hit and then found the cause or the workaround for. Not one of them is in those two pages of docs.

1. `updateDataModel` sent in the same batch as `updateComponents` gets silently dropped. This is the nastiest one. The surface appears normally, nothing errors, and every property bound through `path` comes up empty. Resend the same bytes one second later and the fields fill in immediately. logcat says all three messages parsed OK and the surface count is 1, but the values never land:

```text
parsed OK -> A2uiCreateSurfaceMessage surfaceId=booking
parsed OK -> A2uiUpdateComponentsMessage surfaceId=booking
parsed OK -> A2uiUpdateDataModelMessage surfaceId=booking
activeSurfaces.size=1 ids=[booking]
```

So what I ended up doing: send `createSurface` and `updateComponents` first, wait for the surface to appear, then push the data model. That happens to be the real order a streaming agent produces anyway. Accidental and correct at the same time.

2. catalogId is v0_9, not v0_9_1. The "stable" example on the docs site says `version: v0.9.1` and `.../v0_9_1/...`. The actual constants in the library are `protocol = v0.9` and `.../v0_9/catalogs/basic/catalog.json`. The parser accepts either version string, but `createSurface.catalogId` has to match the id of the catalog you registered exactly. Miss it and the surface never appears, again with no error. My fix: stop copying strings out of docs and read them from the library constants.

3. `processInput` is not a method on `A2uiMessageProcessor`. It is a top-level extension function in the `androidx.a2ui.model.processor` package. You have to `import androidx.a2ui.model.processor.processInput`. Copy the line from the docs verbatim and it will not compile.

4. Failures are silent. Inside, `processInput` swallows `A2uiException`, turns it into `processor.processError(...)`, and emits it only on `outboundEvents`. If you are not collecting that flow (the docs collect it, but never say what skipping it costs), protocol errors and parse failures are completely invisible and all you get on screen is a white box. First thing when debugging this library: wire up `outboundEvents`.

5. `material3-a2ui:1.0.0-alpha01` does not work with the current stable Compose BOM. Its POM demands `androidx.compose.*` 1.13.0-alpha03 and pins material3 to a strict `[1.5.0-alpha28]` range. The stable BOM resolves to 1.12.x, so it will not install. I ended up dropping the BOM and pinning versions by hand.

6. There is no documentation for compileSdk 37 minor 1. `compileSdk = 37` cannot find the already-installed android-37.1. What works is:

```kotlin
compileSdk { version = release(37) { minorApiLevel = 1 } }
```

7. The surface root component id has to be `root`. It is a private constant. Name it anything else and your Column is not the root.

8. Enum values are pure guesswork. Get one wrong and the schema fails, and the docs do not list them:

```text
Text.variant           h1 h2 h3 h4 h5 caption body
Button.variant         primary secondary borderless
TextField.variant      shortText longText number obscured
```

My approach was to `javap` the enums out of the AAR and read them one by one.

One more, unrelated to A2UI but impossible to dodge: adding `org.jetbrains.kotlin.android` under AGP 9.x gets rejected outright. Kotlin support has been built into AGP since 9.0, so you keep only the compose plugin.

## My verdict

It works, and what it renders really is native Material 3, not a wrapped web page. But at alpha the docs are running behind the code. The real catalogId, the shape of `processInput`, message ordering, how to pin versions: you have to unpack the artifacts and look. Good for a proof of concept right now. Do not rush it into production.

On the KMP side, wait. The aar-only fact alone settles it: Compose Multiplatform only gets the Android half.

What I want to try next is closing the outbound path: the user taps Confirm booking, the event goes back to the agent, and the agent sends another `updateComponents` to change the UI based on the result. I only wired that path up. I never actually pressed the button, so I am not claiming here that it works.

Would you let an agent generate the interface in your AI chat client directly? Or would you rather hard-code the UI and only let the agent fill in content? Comments are open.

References:
- A2UI protocol site: https://a2ui.org/
- The A2UI renderer for Jetpack Compose (official docs): https://developer.android.com/develop/ui/compose/agentic
- Custom catalogs: https://developer.android.com/develop/ui/compose/agentic/manage-catalogs
- A2UI Compose release page: https://developer.android.com/jetpack/androidx/releases/a2ui-compose

Verification boundary: both screenshots, the parse logs for the three messages, the catalogId and protocol version constants, the A/B on message ordering, and the build and install all come from running on a real emulator on this machine (Android emulator API 36.1, APK 13,471,211 bytes, BUILD SUCCESSFUL). Component property names and enum values came from `javap` on the unpacked AAR. The tap-to-agent return path was not exercised. Images, video and audio use placeholder renderers, not real loading libraries.
