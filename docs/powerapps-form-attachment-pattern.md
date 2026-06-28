# Power Apps: Form-Based Attachment Upload Pattern

Use this pattern when a screen needs to **upload a file to an existing SharePoint list item**
(i.e., the record already exists — you are patching it, not creating it).

---

## Core Concept

A `Form@2.4.4` bound to an existing SharePoint record can upload attachments via `SubmitForm()`.
The `TypedDataCard@1.0.7` (ClassicAttachmentsEdit variant) inside the form handles the
file-staging and the actual upload to SharePoint. `Patch()` is called inside `OnSuccess`
after the file is safely uploaded.

**Do not use shadow/hidden forms.** A visible form with a single attachment DataCard
works identically and is simpler to maintain.

---

## YAML Structure

```yaml
- formUpload:
    Control: Form@2.4.4
    Variant: Classic
    Layout: Vertical
    Properties:
      DataSource: =YourList
      Item: =gSelectedRecord
      # Edit mode: omit DefaultMode entirely (it is the default)
      # New mode:  DefaultMode: =FormMode.New
      # View mode: DefaultMode: =FormMode.View
      OnFailure: =Notify("Upload failed. Please try again.", NotificationType.Error)
      OnSuccess: |-
        =With(
            {wURL: gSharePointAttachmentBase & Text(gSelectedRecord.ID) & "/" & EncodeUrl(gPendingNewAttName)},
            With(
                {wReq: Patch(YourList, gSelectedRecord, { MyURLColumn: wURL, ... })},
                If(IsBlank(wReq.ID),
                    Notify("Failed to save. Please try again.", NotificationType.Error),
                    Notify("Saved successfully.", NotificationType.Success);
                    Navigate(HomeScreen)
                )
            )
        )
      Visible: =<condition when this upload section is relevant>
      Width: =Parent.Width
    Children:
      - dcUpload:
          Control: TypedDataCard@1.0.7
          Variant: ClassicAttachmentsEdit
          Properties:
            DataField: ="{Attachments}"
            Default: =Blank()          # IMPORTANT — see note below
            DisplayMode: =DisplayMode.Edit
            Update: =attUpload.Attachments
            Width: =Parent.Width
          Children:
            - lblUploadLabel:
                Control: Label@2.5.1
                Properties:
                  Text: ="Attach Document *"
                  Width: =Parent.Width
            - attUpload:
                Control: Attachments@2.3.0
                Properties:
                  Height: =80
                  Items: =Parent.Default   # inherits Blank() → starts empty
                  MaxAttachments: =100
                  Width: =Parent.Width
```

---

## Key Rules

### 1. `Default: =Blank()` on the DataCard — never `=gSelectedRecord.Attachments`

The SharePoint connector determines which files to **delete** by comparing `Default` with the
current control state. If `Default` contains existing attachments and the user does not see
them (because `Items: =Parent.Default` shows nothing useful), those files will be **deleted
on submit**.

Setting `Default: =Blank()` tells the connector "there is nothing to delete" — it will only
upload new files, leaving all existing attachments untouched.

### 2. Validate before `SubmitForm()` — use `CountRows = 0`

Because the control starts empty (`Items: =Parent.Default` where Default is Blank()),
checking whether the user attached a file is simply:

```powerfx
If(
    CountRows(attUpload.Attachments) = 0,
    Notify("Please attach a document", NotificationType.Error),
    Set(gPendingNewAttName, Last(attUpload.Attachments).Name);
    SubmitForm(formUpload)
)
```

### 3. Capture the filename with `Last()` **before** `SubmitForm()`

`Last(attUpload.Attachments).Name` reads the filename from the live control **before** the
form submits. This is safe because the Attachments control appends new files at the end of
its list.

Do **not** use `Last(formUpload.LastSubmit.Attachments).Name` — after submit, SharePoint
returns attachments in **alphabetical order**, so `Last()` on `LastSubmit` picks the wrong
file when multiple attachments exist.

### 4. Construct the attachment URL manually in `OnSuccess`

After `SubmitForm()`, the attachment exists at a predictable SharePoint path.
Build the URL in `OnSuccess` using the filename captured in step 3:

```powerfx
"https://<tenant>.sharepoint.com/sites/<site>/Lists/<ListName>/Attachments/"
    & Text(gSelectedRecord.ID) & "/" & EncodeUrl(gPendingNewAttName)
```

Centralise the base path in `App.OnStart` as a global variable (e.g. `gSharePointAttachmentBase`)
to avoid hardcoding it in every screen.

### 5. Call `Patch()` inside `OnSuccess`, not before `SubmitForm()`

The file does not exist on SharePoint until `SubmitForm()` completes. Any `Patch()` that
writes the URL to the record must happen inside `OnSuccess`, after the upload succeeds.

### 6. Use `With({wReq: Patch(...)}, If(IsBlank(wReq.ID), ...error..., ...success...))` for all writes

This is the standard guard pattern in this codebase — always check `wReq.ID` for blank to
detect a failed write.

---

## Multiple Upload Paths (e.g. Path A / Path B / Path C)

When the same screen has multiple upload scenarios (different forms, different destination
columns), use a global variable `gPendingPath` to route logic in `OnSuccess`:

```powerfx
// In btnSubmit.OnSelect — PATH B
Set(gPendingNewAttName, Last(attShadow_Confirmation.Attachments).Name);
Set(gPendingPath, "B");
SubmitForm(formConfirmation)

// In formConfirmation.OnSuccess
With({wURL: ...},
    Patch(YourList, gSelectedRecord, {
        OrderConfirmationURL: wURL,
        InvoiceMode: {Value: If(gPendingPath = "B", "Deferred", "Direct")},
        ...
    })
)
```

---

## Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| `Default: =gSelectedRecord.Attachments` | User can accidentally delete existing files; worst case all existing attachments are wiped on submit | `Default: =Blank()` |
| `Last(formX.LastSubmit.Attachments).Name` | Returns wrong file when multiple attachments exist (SharePoint alphabetical order) | `Last(attX.Attachments).Name` before submit |
| `Filter(attX.Attachments, IsBlank(AbsoluteUri))` | PA2001 — type checker does not expose Attachments column schema | `CountRows(attX.Attachments) = 0` for check; `Last(attX.Attachments).Name` for name |
| `Patch()` before `SubmitForm()` | URL is constructed before file exists on SharePoint | Move `Patch()` into `OnSuccess` |
| `TypedDataCard` with `PaddingBottom/Left/Right/Top` | PA2108 — unsupported properties on this variant | Remove all Padding properties from TypedDataCard |
| Form inside container "not possible" | Forms **can** be placed inside `GroupContainer@1.5.0` AutoLayout containers | Just place them — no special handling needed |
