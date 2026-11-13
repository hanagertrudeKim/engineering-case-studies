# Building Composable React Forms and Fixing Upload Data Boundaries

**Hana Kim · Front-End Team Lead, BCSD Lab · KOIN**  
**Stack:** React, TypeScript, Ant Design

## At a glance

I built a reusable form system around Ant Design with a compound component API. Screens could assemble inputs, selectors, checkboxes, and uploads under a shared `CustomForm` namespace while choosing their own fields and layout.

An upload issue revealed a separate design constraint: the API and the UI library represented the same images differently. I resolved the mismatch by keeping image URLs as form data and converting them into Ant Design objects for rendering.

Together, these examples show two parts of reusable component design: a clear composition API and explicit boundaries between application data and library-specific state.

## My contribution

My work covered the form component abstraction, its composition API, and the upload integration described in my implementation notes and postmortem. This case study brings those two records together; it does not claim that the compound component pattern itself fixed the upload issue.

## 1. Give forms a consistent composition API

Forms needed familiar controls but different combinations of fields, layout, and domain data. I separated field responsibilities into wrappers around Ant Design and exposed them as properties on `CustomForm`.

The implementation grouped components with `Object.assign(Form, { ... })`. The documented API included text inputs, number inputs, text areas, checkboxes, selectors, single and multiple uploads, buttons, switches, modals, and layout rows.

**Condensed usage example adapted from the original notes:**

```tsx
<CustomForm>
  <CustomForm.Input label="Room name" name="name" />

  <CustomForm.GridRow gridColumns="1fr 1fr">
    <CustomForm.Input label="Room type" name="room_type" />
    <CustomForm.InputNumber label="Room size" name="size" />
  </CustomForm.GridRow>

  <CustomForm.TextArea label="Description" name="description" />
  <CustomForm.MultipleUpload
    domain="lands"
    name="image_urls"
    form={form}
  />
  <CustomForm.Button htmlType="submit">Save</CustomForm.Button>
</CustomForm>
```

`CustomForm` handled a consistent vocabulary for form construction. The consuming screen still decided which fields belonged together. The wrappers carried common field concerns such as labels, names, validation rules, and disabled state.

This was composition over a single component with an expanding list of switches for every screen. It also kept domain choices visible at the call site: a room form could declare its fields directly rather than hiding its structure behind a generic configuration object.

The source demonstrates a namespaced composition API. It does not show a custom React Context implementation, so I do not describe this work as a Context-based state architecture or claim that it eliminated all prop drilling.

## 2. Keep upload data independent of the rendering library

The upload postmortem records request/response type mismatches and HTTP 400/501 responses during implementation. The actionable issue was a disagreement between data representations:

| Boundary | Representation | Purpose |
|---|---|---|
| File selection | Browser file / Ant Design `RcFile` | The file selected for upload |
| Upload request | `FormData` | Send file bytes to the upload API |
| Application form | `string[]` | Store the returned image URLs |
| Upload display | Ant Design `UploadFile[]` | Render existing images and upload status |

The upload API accepted file data and returned a URL. The parent form needed URLs when submitting its payload, while Ant Design needed file-list objects for display. Treating both as one representation made the form and upload component disagree about their values.

I kept the URL list as the form value and derived the display objects from it. After a successful upload, the returned URL was added to the list and assigned to the named form field.

**Illustrative excerpt adapted from the original implementation:**

```ts
// Application data: the form submits image URLs.
const [imageUrls, setImageUrls] = useState<string[]>(
  form.getFieldValue(name) || [],
);

// Presentation data: Ant Design renders UploadFile objects.
const displayFiles: UploadFile[] = imageUrls.map((url, index) => ({
  uid: `existing-${index}`,
  name: url,
  status: "done",
  url,
}));

function recordUploadedUrl(url: string) {
  const nextUrls = [...imageUrls, url];
  setImageUrls(nextUrls);
  form.setFieldValue(name, nextUrls);
}
```

The excerpt omits imports, the upload request, and component wiring. It preserves the documented separation of URL-based form values and derived upload-list objects; it is not a complete production component.

## Why the two decisions belong together

The compound API made a form easy to assemble. The upload adapter made one of its more complicated fields usable without forcing Ant Design's display objects into the application's submission format.

The reusable boundary was therefore more than visual consistency: a screen could compose `CustomForm.MultipleUpload` while keeping its submitted data aligned with the backend contract.

## Results and lessons

The notes demonstrate a composed form API and a concrete fix for the upload representation mismatch. The postmortem reports that separating API-format and UI-format arrays resolved the type mismatch. Neither source includes a measured reuse rate, development-time improvement, or production error reduction, so this case study makes no percentage claim.

The main lesson was to define two contracts separately: **how developers compose a component** and **what data crosses its boundary**. A reusable abstraction needs both.

## Trade-offs and follow-up work

- **Library coupling remains.** The wrappers still depend on Ant Design. A shared namespace makes usage consistent but does not make the implementation library-independent.
- **Local state and form state need coordination.** The historical approach updates both. Reset, deletion, and externally loaded values need deliberate synchronization; the excerpts alone do not establish that every lifecycle path was covered.
- **Async uploads deserve separate review.** The original update uses a captured URL array. Concurrent completions could require a different state-update strategy. That is a follow-up consideration, not a claim that concurrency handling was part of this fix.

## Original documentation

- [Compound Components Pattern — Korean implementation notes](https://github.com/hanagertrudeKim/obsidian-backup/blob/master/%E2%9D%92%20FrontEnd/React/react%20%EC%A0%95%EB%A6%AC/Compound%20Components%20Pattern.md)
- [Ant Design upload component — Korean postmortem](https://github.com/hanagertrudeKim/obsidian-backup/blob/master/%E2%9D%92%20postmortem/antd%20uploadCompoenent.md)

This is a retrospective synthesis of those records. Examples are condensed and translated for readability; historical implementation details and follow-up considerations are distinguished above.

[Back to Engineering Case Studies](README.md) · [Related: API Error Handling and Slack Incident Reporting](koin-error-handling.md)
