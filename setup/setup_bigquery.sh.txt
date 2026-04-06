#!/bin/bash

PROJECT_ID=$(gcloud config get-value project)
DATASET_NAME="maternity_info"
LOCATION="US"

# Generate bucket name if not provided
if [ -z "$1" ]; then
    BUCKET_NAME="gs://maternity-data-$PROJECT_ID"
    echo "No bucket provided. Using default: $BUCKET_NAME"
else
    BUCKET_NAME=$1
fi

echo "----------------------------------------------------------------"
echo "Maternity assitant Setup"
echo "Project: $PROJECT_ID"
echo "Dataset: $DATASET_NAME"
echo "Bucket:  $BUCKET_NAME"
echo "----------------------------------------------------------------"

# 1. Create Bucket if it doesn't exist
echo "[1/5] Checking bucket $BUCKET_NAME..."
if gcloud storage buckets describe $BUCKET_NAME >/dev/null 2>&1; then
    echo "      Bucket already exists."
else
    echo "      Creating bucket $BUCKET_NAME..."
    gcloud storage buckets create $BUCKET_NAME --location=$LOCATION
fi

# 2. Upload Data
echo "[2/5] Uploading data to $BUCKET_NAME..."
gcloud storage cp data/*.csv $BUCKET_NAME

# 3. Create Dataset
echo "[3/5] Creating Dataset '$DATASET_NAME'..."
if bq show "$PROJECT_ID:$DATASET_NAME" >/dev/null 2>&1; then
    echo "      Dataset already exists. Skipping creation."
else    
    bq mk --location=$LOCATION --dataset \
        --description "$DATASET_DESCRIPTION" \
        "$PROJECT_ID:$DATASET_NAME"
    echo "      Dataset created."
fi

# 4. Create Demographics Table
echo "[4/5] Setting up Table: demographics..."
bq query --use_legacy_sql=false \
"CREATE OR REPLACE TABLE \`$PROJECT_ID.$DATASET_NAME.demographics\` (
    zip_code STRING OPTIONS(description='5-digit US Zip Code'),
    city STRING OPTIONS(description='City name, e.g., Los Angeles'),
    neighborhood STRING OPTIONS(description='Common neighborhood name, e.g., Santa Monica, Silver Lake'),
    nurse_name STRING OPTIONS(description='Name of the nurse'),
    pay_per_month STRING OPTIONS(description='payment per month'),
    avaliability STRING OPTIONS(description='Stay in for 24 hrs or Shift service from 8-6 hr'),
	age STRING OPTIONS(description='Age of the nurse' )

)
OPTIONS(
    description='Information of nurse/nannies'
);"

bq load --source_format=CSV --skip_leading_rows=1 --ignore_unknown_values=true --replace \
    "$PROJECT_ID:$DATASET_NAME.demographics" "$BUCKET_NAME/demographics.csv"




# 7. Create Services table
echo "[5/5] Setting up Table: services..."
bq query --use_legacy_sql=false \
"CREATE OR REPLACE TABLE \`$PROJECT_ID.$DATASET_NAME.services\` (
    nurse_name STRING OPTIONS(description='Name of the nurse'),
    services STRING OPTIONS(description='Services offered')
    
)
OPTIONS(
    description='Information of services offered'
);"

bq load --source_format=CSV --skip_leading_rows=1 --replace \
    "$PROJECT_ID:$DATASET_NAME.services" "$BUCKET_NAME/services.csv"

echo "----------------------------------------------------------------"
echo "Setup Complete!"
echo "----------------------------------------------------------------"
