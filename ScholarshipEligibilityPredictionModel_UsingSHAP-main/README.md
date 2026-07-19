# Scholarship Eligibility Predictor

**Live Demo** : file:///C:/Users/dilip/OneDrive/Desktop/Scholarship-eligibility-prediction/index%20(1).html

A machine-learning-powered web application that predicts whether a student is eligible for a scholarship, based on academic performance, financial need, reservation category, location, and age. The system combines a Random Forest classifier with a rule-based scoring engine and per-prediction explainability (SHAP-style feature contributions).

## Features

- **Single prediction**: submit one student's profile and get an eligibility decision with confidence score
- **Batch prediction**: upload a CSV of students and get predictions for all of them at once
- **Explainability**: every prediction includes local feature contributions, showing which factors pushed the decision toward or away from eligibility
- **Configurable scoring**: grade weights and reservation-category bonuses can be overridden per request without retraining the model
- **Academic Score Indicator**: a single 0-100 merit score computed as a weighted average of Class 8, 9, 10, 11, 12 percentages and CGPA
- **Web UI**: a static HTML/CSS/JS front end for interactive use, alongside a full REST API

## Tech Stack

| Layer | Technology |
|---|---|
| ML model | scikit-learn `RandomForestClassifier` |
| Backend API | Flask + Flask-CORS |
| Serving | Gunicorn |
| Data processing | pandas, NumPy |
| Frontend | Static HTML/CSS/JS |
| Model persistence | joblib |

## Project Structure

```
.
├── app.py                    # Flask REST API (prediction service)
├── train_model.py            # Model training script
├── index.html                 # Web UI (static frontend)
├── requirements.txt           # Python dependencies
├── Procfile                   # Process file for deployment (Gunicorn)
├── Scholarshipelegibility.csv # Sample/training dataset
└── model_artifacts/
    ├── random_forest.joblib   # Trained model
    └── model_meta.json        # Model metadata (feature importances, encoders, stats)
```

## How It Works

### 1. Eligibility Score (ground-truth labeling)

Training labels are generated with a rule-based scoring formula, out of a maximum of ~103 points, with an eligibility threshold of 45.

| Component | Max Points | Basis |
|---|---|---|
| Academic merit | 40 | Academic Score Indicator (weighted grade average) |
| Financial need | 30 | Inversely proportional to parents' income (capped at ₹300,000) |
| Reservation category bonus | -10 to +20 | SC/ST: +20, OBC: +12, General: 0, NRI: -10 |
| Location disadvantage | 8 | +8 if rural |
| Age preference | 5 | Younger applicants score slightly higher |

### 2. Academic Score Indicator

A weighted average of six academic inputs (each 0-100):

| Grade | Default Weight |
|---|---|
| Class 8 % | 10% |
| Class 9 % | 10% |
| Class 10 % | 25% |
| Class 11 % | 10-15% |
| Class 12 % | 20% |
| CGPA (normalized) | 20-25% |

Weights are configurable and are normalized to sum to 1.0 if a custom set is supplied.

### 3. Model Training

`train_model.py` does the following:
1. Loads student records from an Excel/CSV source
2. Synthesizes per-grade percentages from CGPA if not already present
3. Computes the Academic Score Indicator and rule-based eligibility label for each student
4. Encodes categorical fields (gender, location, category)
5. Trains a `RandomForestClassifier` (100 trees, max depth 12)
6. Evaluates with accuracy and ROC-AUC
7. Computes sample SHAP-style explanations
8. Saves the model and metadata to `model_artifacts/`

Current model performance (see `model_meta.json`):
- Accuracy: 96.5%
- ROC-AUC: 0.996
- Training samples: 12,000. Test samples: 3,000



### `GET /health`
Health check.
```json
{ "status": "ok", "model_loaded": true }
```

### `GET /model/info`
Returns model metadata: accuracy, feature importances, valid category/gender/location values, default weights and bonuses, and dataset statistics.

### `POST /predict`
Predict eligibility for a single student.

**Request body:**
```json
{
  "gender": "female",
  "location": "rural",
  "category": "OBC",
  "class_8_pct": 78,
  "class_9_pct": 80,
  "class_10_pct": 82,
  "class_11_pct": 79,
  "class_12_pct": 85,
  "CGPA_pct": 84,
  "parents_income": 120000,
  "age": 19,
  "weights": {
    "class_8": 0.10, "class_9": 0.10, "class_10": 0.25,
    "class_11": 0.15, "class_12": 0.20, "CGPA": 0.20
  },
  "category_bonuses": { "OBC": 12 }
}
```

**Response:**
```json
{
  "eligible": true,
  "confidence": 91.4,
  "probability": { "eligible": 0.914, "not_eligible": 0.086 },
  "academic_score": 81.35,
  "eligibility_score": 67.2,
  "weights_used": { "...": "..." },
  "category_bonuses_used": { "...": "..." },
  "shap": {
    "all_features": [ { "feature": "Category", "contribution": 0.20, "direction": "positive" } ],
    "top_positive_factors": [ "..." ],
    "top_negative_factors": [ "..." ],
    "dominant_feature": "Category"
  },
  "narrative": "This student is predicted ELIGIBLE with 91% confidence. ...",
  "input_received": { "...": "..." }
}
```

`weights` and `category_bonuses` are optional. Omit them to use the model's defaults.

### `POST /batch_predict`
Upload a CSV (`multipart/form-data`, field name `file`) with columns:

```
gender, location, category, class_8_pct, class_9_pct, class_10_pct,
class_11_pct, class_12_pct, CGPA_pct, parents_income, age
```

Optional form fields `weights` and `category_bonuses` (JSON strings) apply the same overrides to every row.

Returns a summary plus a per-row prediction array.

## Getting Started

### Prerequisites
- Python 3.10+
- pip

### Installation
```bash
git clone <your-repo-url>
cd <your-repo-name>
pip install -r requirements.txt
```

### Train the model
```bash
python train_model.py
```
This generates `model_artifacts/random_forest.joblib` and `model_artifacts/model_meta.json`.

A pre-trained model is already included in this repo, so this step is optional unless you want to retrain on new data or different weights.



### Open the web UI
Open `index.html` in a browser, or serve it via any static file host. Point it at your running API instance.

## Deployment

This project includes a `Procfile` for platforms like Heroku or Render:
```
web: gunicorn app:app
```

Make sure the trained model artifacts (`model_artifacts/random_forest.joblib` and `model_artifacts/model_meta.json`) are present in the deployment environment. Either commit them to the repo or run `train_model.py` as part of your build/release step.

## Dataset

The sample dataset (`Scholarshipelegibility.csv`) contains anonymized student records with the following fields: `first_name`, `last_name`, `email`, `gender`, `Location`, `Category`, `CGPA`, `Parents_Income`, `Age`, `Adhaar_Number`.