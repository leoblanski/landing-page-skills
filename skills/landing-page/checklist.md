# Phase gates

Tick every box before moving on. A box you cannot tick is a finding to fix,
not a box to skip.

## 1. Context
- [ ] `.agents/product-marketing.md` exists and matches PRODUCT.md (price, audience, proof numbers)
- [ ] Audience stated, including how technical they are and which device paid traffic arrives on

## 2. Offer and funnel
- [ ] One primary action named
- [ ] What is free vs paid decided; price shown only where the offer skill places it
- [ ] Upsell or bump decision recorded (or explicitly none)

## 3. Structure
- [ ] Above the fold: H1, one subtitle, the primary action, nothing else interactive
- [ ] Primary action above the fold at 360x780 and 390x844, reachable by thumb
- [ ] Paid-traffic page: no site navbar, no exits; footer keeps legal links and contact
- [ ] Sections in the order the cro skill gives, each with its job in one line
- [ ] Form: visible label, placeholder is an example, correct input attributes

## 4. Copy
- [ ] Every string passes copywriting; no AI-sounding patterns ("Do X ao Y", "não é X, é Y", "desbloqueie", "transforme", stacked rhetorical questions)
- [ ] No em dashes; pt/en parity
- [ ] Claims list with a source for each (query, constant, doc)
- [ ] FAQ built from real objections, not invented ones
- [ ] Meta personal-attributes policy respected in anything reused in ads

## 5. Proof and media
- [ ] Product shown with real captures, never AI-generated UI
- [ ] Device mockups match real proportions (iPhone 15 body 0.485 w/h, Dynamic Island, status bar); desktop in a real browser frame
- [ ] Each mockup has callouts pointing at the feature it sells
- [ ] Testimonials and photos only from consented, approved sources
- [ ] Media lazy-loaded, sized, reduced-motion shows a still

## 6. Visual identity
- [ ] Only project tokens, fonts, components; nothing added at this phase
- [ ] One column/grid, true centering measured, consistent vertical rhythm, no orphans
- [ ] Primary action is the focal point by size, contrast and whitespace

## 7. SEO and AI search
- [ ] Prerendered HTML has title <= 60, description, canonical, one h1 with the main term
- [ ] One FAQPage in sync with the visible FAQ, answers present in the HTML
- [ ] hreflang/canonical correct for BR-only pages; no global JSON-LD contradicting the page
- [ ] A self-contained citable "what it is" passage
- [ ] Not cannibalizing another page's queries; internal links in

## 8. Measurement
- [ ] Funnel events fire once with useful params; purchase counted once (server)
- [ ] Attribution (utm, gclid, fbclid) captured before any URL rewrite and stored with the conversion
- [ ] Ad data sent to Meta/Google only with cookie consent
- [ ] Dashboard of the funnel readable from the database

## 9. QA
- [ ] Full journey walked at 1440, 1024, 768, 430, 390, 360 including every error and payment state
- [ ] CLS < 0.1 and LCP measured on mobile throttled
- [ ] Real path exercised end to end (not only mocks): bot check, payment webhook, delivery

## 10. Review
- [ ] Code review PASS
- [ ] Compliance report: each skill above, each rule, how the page complies
- [ ] Only then show the founder, with screenshots
