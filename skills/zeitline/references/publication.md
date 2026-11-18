# Publication and recovery

## Owner review

Open the hosted tracker while signed in. In **Share → Public content review**, inspect title, text, identities, links and media, choose the names policy, supply the inventory needed for review and check the confirmation. The initials policy removes links and media. If both the source's sharing permissions and public publication need changing, save source access first. Then choose **Publish to web** or **Update public snapshot**. The Share action saves a review against the current content, validates, and publishes if the report passes.

An API validation returns `blocked_report` with no receipt when the review is missing; no MCP or CLI tool records the review, so the first publish is a browser action. A previously reviewed snapshot can need another owner review when outward identities, URLs or prose change. On a blocker, use the browser or repair the source and validate again. Only a browser session can approve permitted validation exceptions.

## API-key MCP

OAuth sessions cannot call these tools. Use a general key in `ZEITLINE_API_KEY` with `zeitline:read` and `zeitline:write`. See [Scheduled writes](scheduling.md) for the authenticated MCP transport.

1. `validate_publication` with `{zeitlineId}`. Include `workspaceId` if validation is scoped to a workspace. Require `publicationValid: true` and a passing `status` (`pass` or `pass_with_exceptions`). Save `receipt.receiptId`.
2. `publish_zeitline` with `{zeitlineId, validationReceipt: "<receipt-id>", allowComments: false, showInGallery: false}`. Omit `slug` to use the reviewed slug or Zeitline's automatic one. If the owner chose a custom slug, it belongs in the browser review. Supply the intended IANA `timeZone` for relative snapshot periods. Keep options fixed when retrying the exact receipt. Validate again for a new revision or an expired receipt.
3. Verify the returned `publicUrl` in a signed-out browser. The response includes both source and snapshot revisions. The slug stays useful after republishing. A private update needs its own validation and publish to change the public snapshot.

## API-key REST

Send HTTPS to `https://zeitline.com` with `Authorization: Bearer …` from the env var and `Content-Type: application/json`. Read both the HTTP status and the response body.

| Operation | Method and path | Body and receipt |
| --- | --- | --- |
| Validate | `POST /api/v1/zeitlines/<id>/publication-validation` | `{}` or `{"workspaceId":"<workspace-id>"}`. Proceed only on `report.status` of `pass` or `pass_with_exceptions`. Record `receipt.receiptId`. |
| Publish | `POST /api/v1/zeitlines/<id>/publish` | `{"validationReceipt":"<receipt-id>","allowComments":false,"showInGallery":false}`. Include `timeZone` for relative snapshot periods. Read `publicUrl` and `snapshotRevision`. |
| Unpublish | `POST /api/v1/zeitlines/<id>/unpublish` | Use `{}` when the owner requests removal. Confirm from a signed-out browser. The browser Share dialog also offers **Unpublish**. |

For X, share the ordinary `/view/<slug>` link. For a website, use the iframe from the skill. Use no secret in any viewer URL.
