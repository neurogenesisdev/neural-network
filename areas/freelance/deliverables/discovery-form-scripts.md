---
type: template
status: active
owner: mentor
created: 2026-09-26
updated: 2026-09-26
tags: [template, discovery, client]
---

# Discovery form scripts (Google Apps Script)

> [!NOTE] Summary
> Scripts that build the discovery forms of [[discovery-inventory]] in Google
> Forms, each with a linked answers Sheet. Run at script.google.com → New
> project → paste → Run → approve (unverified-app warning is normal: Advanced →
> Go to project). Links appear in the execution log. Re-running creates a new
> form; delete old ones and their "(responses)" Sheets. Per client, only the
> `sites` / `explore` lists of the reference script change. The materials form
> is built by hand after the call (Apps Script can't create file-upload
> questions).

## After every build

- Open the edit link → **Publish** if a Publish button shows.
- Settings → Presentation: "Disable autosave for all respondents" is **off**.
- Preview on a phone; test a draft phone → laptop, then **Clear form**.
- Send the short link (cut everything after `viewform`).
- Delete test submissions in the form **and** their row in the linked Sheet (deleting a response doesn't remove the Sheet row).
- Settings → Responses: check **Collect email addresses = Verified** (the script sets it where the API allows) and set **Send responders a copy = When requested** by hand (no script setting for it, as far as known).

Standard settings for every discovery form (luis, 2026-09-26): verified email, response editing on, limit to 1 response on (so edits replace instead of duplicating; needs sign-in, which autosave needs anyway), "submit another response" link off, progress bar on, autosave not disabled, custom confirmation message.

## Questions form

```javascript
function buildQuestionsForm() {
  const DETAILS = "Details, or why you're not sure yet (optional)";

  const intro = [
    "This short form covers the basics of your website: what it's for, who it's for, and what it needs. It takes about 10–15 minutes.",
    "",
    "- Pick from the options where you can. If none fits, use \"Other\".",
    "- \"Not sure yet\" is always a fine answer. The text field under a question is optional: use it for details, or to say why you're not sure.",
    "- Your progress saves automatically while you're logged into your Google account, for 30 days after your last change. You can continue later on your phone or laptop. When you come back, the form opens at the start with your answers kept, so just click Next to get back to where you stopped.",
    "- Anything that's still open, we'll go through together in a short call."
  ].join("\n");

  const form = FormApp.create("Your website: the basics");
  form.setDescription(intro)
      .setProgressBar(true)
      .setLimitOneResponsePerUser(true)
      .setAllowResponseEdits(true)
      .setShowLinkToRespondAgain(false)
      .setConfirmationMessage("Thank you! I'll go through your answers and get back to you to set up a short call.");
  if (FormApp.EmailCollectionType) form.setEmailCollectionType(FormApp.EmailCollectionType.VERIFIED);

  const details = (title) => form.addParagraphTextItem().setTitle(title || DETAILS);

  // 1 of 4 — Why and who
  form.addPageBreakItem().setTitle("1 of 4: Why and who");

  form.addGridItem()
      .setTitle("1. What is your website for?")
      .setHelpText([
        "Rate what matters to you. Tick as many as you like, and skip anything that doesn't apply.",
        "- Get booked or hired: classes, sessions, projects, calls",
        "- Sell my own products: physical or digital",
        "- Show my work: portfolio, gallery",
        "- Professional home base: who I am, all in one place",
        "- Grow my audience: followers, subscribers, newsletter",
        "- Earn from recommendations: affiliate links, discount codes"
      ].join("\n"))
      .setRows([
        "Get booked or hired",
        "Sell my own products",
        "Show my work",
        "Professional home base",
        "Grow my audience",
        "Brand partnerships",
        "Earn from recommendations",
        "Be found on Google"
      ])
      .setColumns(["High", "Medium", "Low"]);
  details("Tell me more: what exactly should people book, buy, follow or contact you about? (optional)");

  form.addCheckboxItem()
      .setTitle("2. Who visits your website?")
      .setHelpText("Tick all that apply.")
      .setChoiceValues([
        "Clients or customers",
        "Students or class participants",
        "Followers and fans",
        "Brands and collaboration partners",
        "Employers or recruiters",
        "Press",
        "Friends and family",
        "Not sure yet"
      ])
      .showOtherOption(true)
      .setRequired(true);
  details("Describe your most important visitor in a sentence (optional)");

  // 2 of 4 — What
  form.addPageBreakItem().setTitle("2 of 4: What");

  form.addGridItem()
      .setTitle("3. What do you already have?")
      .setRows([
        "A domain (web address)",
        "A logo",
        "Brand colours or fonts",
        "An existing website",
        "Professional photos",
        "Written texts",
        "Social media accounts",
        "A physical location"
      ])
      .setHelpText("Written texts = bio, descriptions of what you offer. Physical location = studio, shop or office.")
      .setColumns(["Yes", "No", "Not sure"])
      .setRequired(true);
  details("Links or details, e.g. your current website or domain name (optional)");

  form.addCheckboxItem()
      .setTitle("4. Which features should your website have?")
      .setHelpText("Tick all that apply.")
      .setChoiceValues([
        "Booking (people book classes, sessions or calls)",
        "Online payment (for bookings, courses or products)",
        "Shop (sell physical or digital products)",
        "Gallery or portfolio (show your work)",
        "Blog, articles or recipes (regular posts)",
        "Newsletter signup",
        "Videos from YouTube or Instagram shown on the site",
        "Brand and partner inquiries (collaboration page)",
        "Recommended products (links or discount codes you earn a commission on)",
        "Not sure yet"
      ])
      .showOtherOption(true)
      .setRequired(true);
  details("Seen something on another website you'd want? Describe it or paste the link (optional)");

  // 3 of 4 — How
  form.addPageBreakItem().setTitle("3 of 4: How");

  form.addCheckboxItem()
      .setTitle("5. How should visitors reach you?")
      .setHelpText("Tick all that apply.")
      .setChoiceValues([
        "Email",
        "Contact form",
        "Booking link (e.g. Calendly)",
        "Instagram DM",
        "WhatsApp",
        "Phone",
        "Not sure yet"
      ])
      .showOtherOption(true)
      .setRequired(true);
  details("The email address, handles or links to show on your site (optional)");

  form.addCheckboxItem()
      .setTitle("6. Which language(s) should your website be in?")
      .setChoiceValues(["English", "German", "Not sure yet"])
      .showOtherOption(true)
      .setRequired(true);
  details();

  form.addGridItem()
      .setTitle("7. How should your website feel?")
      .setHelpText([
        "For each pair, 1 is the left word, 5 the right word, 3 means no preference.",
        "- Calm – Bold: quiet, soft colours or strong colours and energy.",
        "- Minimal – Rich: little on each page or lots to explore.",
        "- Professional – Personal: business-like or warm and about you.",
        "- Light – Dark: bright backgrounds or dark ones."
      ].join("\n"))
      .setRows(["Calm – Bold", "Minimal – Rich", "Professional – Personal", "Light – Dark"])
      .setColumns(["1", "2", "3", "4", "5"])
      .setRequired(true);
  details("Anything to add about the look? (optional)");

  form.addParagraphTextItem()
      .setTitle("8. Ideas for your web address")
      .setHelpText("Up to 3 name ideas, one per line, in case your first choice is taken. Don't buy anything yet.");
  form.addCheckboxItem()
      .setTitle("Preferred ending")
      .setChoiceValues([".com", ".de", ".net", ".me", "No preference"])
      .showOtherOption(true)
      .setRequired(true);

  // 4 of 4 — When and formalities
  form.addPageBreakItem().setTitle("4 of 4: When and formalities");

  form.addDateItem()
      .setTitle("9. Does your website need to be ready by a certain date?")
      .setHelpText("Leave empty if there's no deadline.");
  form.addParagraphTextItem().setTitle("What's the date for? (optional)");

  form.addSectionHeaderItem()
      .setTitle("10. Impressum")
      .setHelpText("German law requires most websites to show the owner's name and a postal address (the \"Impressum\"). A P.O. box isn't enough.");
  form.addMultipleChoiceItem()
      .setTitle("Which address can we use?")
      .setChoiceValues([
        "My own address",
        "c/o a friend or family member",
        "A paid address service (about €5–15 a month)",
        "Not sure yet"
      ])
      .setRequired(true);
  form.addMultipleChoiceItem()
      .setTitle("Are you registered as self-employed (a registered business or freelancer)?")
      .setChoiceValues(["Yes", "No", "Not sure yet"])
      .setRequired(true);
  form.addTextItem().setTitle("VAT ID (USt-IdNr.), if you have one (optional)");
  details();

  form.addParagraphTextItem().setTitle("Anything else you want to tell me? (optional)");

  const sheet = SpreadsheetApp.create("Your website: the basics (responses)");
  form.setDestination(FormApp.DestinationType.SPREADSHEET, sheet.getId());
  if (typeof form.setPublished === "function") form.setPublished(true);

  Logger.log("Edit: " + form.getEditUrl());
  Logger.log("Send to client: " + form.getPublishedUrl());
  Logger.log("Answers sheet: " + sheet.getUrl());
}
```

## Reference form

Per client: replace `sites` (area, name, URL; 3–5 per area) and `explore` (unrated extras). Area counters ("Yoga 2 of 4") assume 4 sites per area; adjust the `of 4` text if an area has a different count.

```javascript
function buildReferenceForm() {
  const intro = [
    "This form helps shape the design and content of your website. Instead of describing what you want from scratch, you react to real websites.",
    "",
    "How it works",
    "- Each section is one website. Open the link, look around for a minute or two, then come back to this tab.",
    "- Give two main ratings: Style and Content. The next page explains what each rating means.",
    "- Below them are a few optional ratings. Only fill those in if something stands out.",
    "- Only write a comment if you really like or dislike something. A short note is enough.",
    "",
    "Your progress saves automatically while you're logged into your Google account, for 30 days after your last change. You can stop anytime and continue later, on your phone or laptop. When you come back, the form opens at the start with your answers kept, so just click Next to get back to where you stopped.",
    "",
    "At the end there's a list of more websites to browse if you're curious. No rating needed there."
  ].join("\n");

  const guide = [
    "Read this once before you start. You can come back to this page with the Back button.",
    "",
    "MAIN RATINGS (required)",
    "Style: how it looks and feels.",
    "Content: what it offers and says.",
    "",
    "OPTIONAL RATINGS (only if something stands out)",
    "Style",
    "- Mood: colours, photos and overall feel. The first impression.",
    "- Layout: spacing, fonts and how much text is on screen. Calm or cramped?",
    "- Ease: how easily you find your way around and find what you're looking for.",
    "Content",
    "- Offer: how quickly it's clear what the person does and how to book or buy.",
    "- Voice: how the site speaks. Personal or formal, warm or cool?",
    "- Personality: how much of the person comes through. Their story, photos of them, their life."
  ].join("\n");

  const sites = [
    // ["Area", "Site name", "https://..."],
  ];

  const explore = [
    // "Site name: https://...",
  ];

  const scale = ["Like", "Neutral", "Dislike"];
  const dims = ["Mood", "Layout", "Ease", "Offer", "Voice", "Personality"];

  const form = FormApp.create("Website references");
  form.setDescription(intro)
      .setProgressBar(true)
      .setLimitOneResponsePerUser(true)
      .setAllowResponseEdits(true)
      .setShowLinkToRespondAgain(false)
      .setConfirmationMessage("Thank you! This helps a lot. I'll go through your ratings and we'll talk about them in our call.");
  if (FormApp.EmailCollectionType) form.setEmailCollectionType(FormApp.EmailCollectionType.VERIFIED);

  form.addPageBreakItem().setTitle("How to rate").setHelpText(guide);

  const counters = {};
  sites.forEach(([area, name, url]) => {
    counters[area] = (counters[area] || 0) + 1;
    form.addPageBreakItem()
        .setTitle(`${area} ${counters[area]} of 4: ${name}`)
        .setHelpText(url);
    form.addMultipleChoiceItem().setTitle("Style").setChoiceValues(scale).setRequired(true);
    form.addMultipleChoiceItem().setTitle("Content").setChoiceValues(scale).setRequired(true);
    form.addGridItem()
        .setTitle("Optional: anything that stands out?")
        .setRows(dims)
        .setColumns(scale);
    form.addParagraphTextItem().setTitle("Comment (optional)");
  });

  form.addPageBreakItem()
      .setTitle("Explore more")
      .setHelpText("More websites to browse whenever you're curious. No rating needed.\n\n" + explore.join("\n"));
  form.addParagraphTextItem().setTitle("Anything else you want to tell me?");

  const sheet = SpreadsheetApp.create("Website references (responses)");
  form.setDestination(FormApp.DestinationType.SPREADSHEET, sheet.getId());
  if (typeof form.setPublished === "function") form.setPublished(true);

  Logger.log("Edit: " + form.getEditUrl());
  Logger.log("Send to client: " + form.getPublishedUrl());
  Logger.log("Answers sheet: " + sheet.getUrl());
}
```

Chantal's filled-in site lists: [[../clients/chantal-fuelber/reference-addendum]].
