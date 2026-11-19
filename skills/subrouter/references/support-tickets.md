# Support ticket message images

The source contract supports optional image attachments on ticket replies. Verify the target deployment supports this addition before using it; this document does not assert that a rollout has completed.

Upload PNG/JPG/WebP files with `POST /api/report/upload-image`, using multipart field `file` and the existing authenticated management API client. Each file must be at most 5 MB, and each message can contain at most 3 images. Read `data.urls` from the successful upload response.

| Sender | Endpoint | JSON body |
| --- | --- | --- |
| Reporting user | `POST /api/report/:id/reply` | `{"content":"More details","evidence":["/api/report/evidence/mre_..."]}` |
| Merchant | `PUT /api/provider/reports/:id/reply` | `{"reply":"Merchant response","evidence":["/api/report/evidence/mre_..."]}` |
| Platform, ordinary ticket | `PUT /api/admin/provider/reports/:id` | `{"admin_remark":"Platform response","evidence":["/api/report/evidence/mre_..."]}` |
| Platform, fraud ticket | `PUT /api/admin/provider/fraud-reports/tickets/:id` | Same body as ordinary platform replies, with fraud management permission. |

Text can be empty when images are attached. Only the sender's own unused uploads can be attached. Sending commits the message and attachment binding together. Do not reuse a successfully sent attachment for another message, or combine a platform image reply with compensation or deposit penalties.

Ticket details return image metadata under each `replies[].evidence` entry, including `id`, `actor_type`, `content_type`, and `created_time`. Download bytes using authenticated `GET /api/report/evidence/:id`; these are private images, not public links. Merchant access still requires the ticket to be visible to that merchant. A fraud ticket remains private until released by the platform.

Message images are distinct from platform investigation evidence. Platform reply images do not satisfy the evidence requirement for releasing a fraud ticket, and remain available as conversation attachments after release.
