# maternity_assistant
This hosts the code for maternity assistant multi ai agent

Steps to deploy and run


Login to GCP , select your project

gcloud auth list

gcloud config get project

export PROJECT_ID=$(gcloud config get project)

gcloud auth application-default login

Go to your directory and run setup scripts

cd /home/shilpa_s4g/maternity_assistant

chmod +x setup/setup_env.sh

./setup/setup_env.sh

chmod +x setup/setup_bigquery.sh

./setup/setup_bigquery.sh

python3 -m venv .venv

source .venv/bin/activate

cd adk_agent/maternity

pip install -r requirements.txt

source .env

Creating service account.
gcloud iam service-accounts create ${SA_NAME} \
  --display-name="Service Account for cloud run "

gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
  --member="servceAccount:$SERVICE_ACCOUNT" \
  --role:"roles/aiplatform.user"
  
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
  --member="servceAccount:$SERVICE_ACCOUNT" \
  --role:"roles/mcp.toolUser"

gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
  --member="servceAccount:$SERVICE_ACCOUNT" \
  --role:"roles/bigquery.jobUser"

gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
  --member="servceAccount:$SERVICE_ACCOUNT" \
  --role:"roles/bigquery.dataViewer"

gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
  --member="servceAccount:$SERVICE_ACCOUNT" \
  --role:"roles/bigquery.user"

Run the deployment

adk deploy cloud_run  
  --project=$GOOGLE_CLOUD_PROJECT \\  
  --region=us-central1 \\  
  --service_name=maternity_asssitant \\  
  --with_ui \\  
  . \\  
  -- \\ 
  --lables=cohort-1=hackthon-adk \\  
  --set-env-vars="ADK_ALLOW_ORIGINS='regex:https://.*\.cloudshell\.dev'"


  

