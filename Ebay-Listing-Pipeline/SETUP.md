# RarezWarez AI Inventory Pipeline — Setup Guide

This bundle contains everything needed to deploy your own copy of the pipeline.
No credentials or account-specific IDs are included — you provide your own via
environment variables in step 6.

## What's in this bundle
- `main.py` — the Flask app that runs on Cloud Run
- `Dockerfile` — container build instructions
- `requirements.txt` — Python dependencies
- `.gitignore` — keeps local junk (caches, `.env`) out of your own repo

## Setup steps

- **Install prerequisites**: [Google Cloud CLI](https://cloud.google.com/sdk/docs/install), then run `gcloud init` and log in to your own GCP project.
- **Enable required APIs**:
  ```
  gcloud services enable run.googleapis.com cloudbuild.googleapis.com \
    cloudscheduler.googleapis.com storage.googleapis.com \
    sheets.googleapis.com generativelanguage.googleapis.com
  ```
- **Create two Cloud Storage buckets** — one for incoming raw photos, one for finished listing zips:
  ```
  gsutil mb gs://YOUR-RAW-BUCKET-NAME
  gsutil mb gs://YOUR-OUTPUT-BUCKET-NAME
  ```
  If you want the zip links to be clickable straight from the sheet, make the output bucket (only the output bucket) publicly readable:
  ```
  gsutil iam ch allUsers:objectViewer gs://YOUR-OUTPUT-BUCKET-NAME
  ```
- **Get a Gemini API key** from [Google AI Studio](https://aistudio.google.com/apikey).
- **Create a Google Sheet** for the inventory log, with a header row:
  `Timestamp | Title | Condition | Description | Estimated_Value | Estimated_Shipping | Zip_URL | Comps_URL`
  Copy its ID out of the URL (`docs.google.com/spreadsheets/d/THIS_PART/edit`) — you'll need it in step 6. **Keep its sharing setting on "Restricted"** — only share it with the service account below, not "anyone with the link."
- **Give your Cloud Run service account access**: share the Sheet (Editor) with the service account's email (`PROJECT_NUMBER-compute@developer.gserviceaccount.com` by default, or a dedicated service account if you created one), and grant it `Storage Object Admin` on both buckets.
- **Deploy to Cloud Run** from this folder:
  ```
  gcloud run deploy rarezwarez-pipeline --source . --region YOUR-REGION \
    --memory 2Gi --timeout 900 \
    --set-env-vars GEMINI_API_KEY=YOUR_GEMINI_KEY,SHEET_ID=YOUR_SHEET_ID,BUCKET_NAME=YOUR-RAW-BUCKET-NAME,OUTPUT_BUCKET_NAME=YOUR-OUTPUT-BUCKET-NAME
  ```
- **Schedule the batch job** to poll every minute:
  ```
  gcloud scheduler jobs create http rarezwarez-tick \
    --schedule="* * * * *" \
    --uri="YOUR_CLOUD_RUN_URL/run-batch" \
    --http-method=GET \
    --location=YOUR-REGION
  ```
- **Point your photo-upload app** (e.g. FolderSync on Android, or any tool that can auto-upload a camera folder) at `gs://YOUR-RAW-BUCKET-NAME`.
- **Test it**: take a few photos of an item, then cover the lens completely for one photo (the "blackout divider"), then photograph the next item. Within about a minute of the divider photo finishing its upload, a new row should appear in your Sheet.
- **Reset the sheet anytime** by visiting `YOUR_CLOUD_RUN_URL/reset-sheet` (clears all rows, keeps headers).

## Notes
- Each `/run-batch` tick processes one item and returns — a 200-photo batch takes roughly as many minutes as it has items, not 15 minutes flat. Scale `--timeout` down if you don't need the extra headroom.
- The blackout-divider brightness/uniformity thresholds in `main.py` (`is_blackout_photo`) were tuned for one specific phone camera's auto-exposure behavior. Watch the `BLACKOUT_SCAN` lines in Cloud Run logs after your first few batches and adjust the thresholds if dividers are being missed or item photos are being misread as dividers.
