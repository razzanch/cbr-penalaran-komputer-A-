# Case-Based Reasoning on Indonesian Court Decisions
An end-to-end pipeline for building a CBR system using Mahkamah Agung decisions.

## Table of Contents
- [Overview](#overview)
- [Installation](#installation)
- [Project Structure](#project-structure)
- [Pipeline: Step-by-Step](#pipeline-step-by-step)
  - [Stage 1: Build Case Base](#stage-1-build-case-base)
  - [Stage 2: Case Representation](#stage-2-case-representation)
  - [Stage 3: Case Retrieval](#stage-3-case-retrieval)
  - [Stage 4: Solution Reuse](#stage-4-solution-reuse)
  - [Stage 5: Evaluation](#stage-5-model-evaluation)
- [License & Credits](#license-credits)

## 1. Overview
This repository implements a complete Case-Based Reasoning (CBR) pipeline on Indonesian court rulings with a focus on:

- Corpus collection and cleaning from Mahkamah Agung RI

- Structured representation of court decisions

- Retrieval using two approaches:
  TF-IDF + Cosine Similarity (unsupervised) and TF-IDF + SVM (supervised classification)

- Reusing solutions from the most relevant past cases

- Performance evaluation using standard metrics (Accuracy, Precision, Recall, F1-Score)

### Selected case domain:
**General Criminal – Murder Cases from Pengadilan Negeri Medan**

## 2. Installation
Start by cloning and setting up the environment:

```bash
git clone https://github.com/razzanch/cbr-penalaran-komputer-A-.git
cd cbr-penalaran-komputer-A-

python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

```

## 3. Project Structure

```bash
├── pdf_downloaded/          # Raw MA PDF Document downloads
├── data/
│   ├── raw/                 # Cleaned text files
│   ├── processed/           # .csv/.json representations and features
│   ├── eval/                # Ground truth queries, metrics, errors
│   └── results/             # Predictions & evaluation outputs
├── logs/                    # Cleaning logs
├── 03_retrieval_model.pkl   # Pretrained retrieval model
├── 03_vectorizer.pkl        # TF-IDF vectorizer
├── cbr_law.ipynb            # Jupyter notebook for CBR law analysis
├── README.md                # This guide
└── requirements.txt         # Python dependencies
```


## 4. Pipeline: Step-by-Step

### Stage 1 – Build Case Base
#### Objective

This stage aims to build the initial case base by collecting, extracting, and cleaning court decision documents from the official source of the Supreme Court of the Republic of Indonesia. The final result of this stage is cleaned decision texts that are ready to be further processed in the Case-Based Reasoning (CBR) cycle.

---

#### Work Steps

##### 1. Document Selection and Download

- Case domain selected: **General Criminal – Murder (PN MEDAN)**
- Source of documents: Supreme Court of the Republic of Indonesia Decision Directory
- Document format: PDF
- Number of documents: **47 documents**

The documents are manually downloaded and stored in the `pdf_downloaded/` folder.

---

##### 2. Conversion and Text Extraction

Each PDF file is converted into plain text using the `pdfminer` library. The purpose of this conversion is to obtain the content of the decision in a format that can be processed further.

---

##### 3. Text Cleaning

The extracted text is cleaned by:

- Removing watermarks, headers, footers, and page numbers
- Removing disclaimer content from the Supreme Court of Indonesia
- Normalizing spaces and characters (lowercase)
- Calculating the document integrity ratio (clean text length compared to the original text length)

Documents are only stored if they meet the minimum integrity ratio requirement of ≥ 80%.

---

##### 4. Validation and Logging

All documents are logged in the cleaning process log (`logs/cleaning.log`) with information about the integrity ratio for each case. This log helps monitor the data quality and detect problematic documents.

---

#### Outputs of This Stage

- The folder `/data/raw/*.txt` contains 47 cleaned decision text files that passed validation.
- The log file `/logs/cleaning.log` contains the integrity validation records for each case.
- All documents have an integrity ratio above 88%, indicating that the extraction and cleaning process has been successfully carried out.

Example of validation log:
[OK] case_001 processed (89.08% valid)  
[OK] case_002 processed (89.29% valid)  
[OK] case_003 processed (89.45% valid)  
...  
[OK] case_047 processed (89.34% valid)

---

This first stage successfully prepares a set of cases with clean text quality suitable for use as the case base in the CBR system. This stage serves as an important foundation for the representation and retrieval processes in the following stages.

### Stage 2 – Case Representation
#### Objective

This stage aims to represent each decision in an organized data structure. The result of this representation becomes a structured database ready to be used for retrieval and further analysis in the Case-Based Reasoning (CBR) system.

---

#### Work Steps

##### 1. Metadata Extraction

Each cleaned document is analyzed to extract important information as metadata, including:

- Case Number (`no_perkara`)
- Decision Date (`tanggal`)
- Fact Summary (`ringkasan_fakta`)
- Article(s) charged (`pasal`)
- Related parties (Defendant and Victim)
- Full text (`text_full`)

Extraction is carried out using a pattern-based approach (regex) on the document content.

---

##### 2. Storing Structured Data

The extracted data is stored in two formats:

- **CSV**: `data/processed/cases_extracted.csv`
- **JSON**: `data/processed/cases_extracted.json`

The column structure used includes:

- `case_id`
- `no_perkara`
- `tanggal`
- `ringkasan_fakta`
- `pasal`
- `pihak`
- `text_full`

Number of cases successfully processed: **47 cases**

Example terminal output:

[SUCCESS] 47 cases saved to:

CSV → data/processed/cases_extracted.csv

JSON → data/processed/cases_extracted.json

---

##### 3. Feature Engineering

To enhance the use of case data, feature engineering is performed, which includes:

- **Word Count (Length)**: Counting the total number of tokens (words) in the text.
- **Bag-of-Words (BoW)**: Counting word frequencies for each case.
- **Simple QA-Pairs**: Generating question and answer pairs from the text content.

QA-Pairs include example questions such as:

- What is the case number?
- What article was violated?
- Who is the defendant?
- Who is the victim?

---

##### 4. Storing Features

The results of feature engineering are stored in JSON format:

- `data/processed/features_length.json`
- `data/processed/features_bow.json`
- `data/processed/features_qa_pairs.json`

Example terminal output:

[SUCCESS] Feature Engineering completed!

Length saved to: data/processed/features_length.json

Bag-of-Words saved to: data/processed/features_bow.json

QA-pairs saved to: data/processed/features_qa_pairs.json

---

The representation stage successfully created an organized data structure for 47 cases. Each case is equipped with metadata, fact summaries, and additional features to support the retrieval and prediction processes in the next stages of the CBR system.

### Stage 3 – Case Retrieval
#### Objective

This stage aims to find the most relevant and similar old cases to the new case query submitted. This process is the core part of the Case-Based Reasoning (CBR) system to support legal precedent analysis and search.

---

#### Work Steps

##### 1. Vector Representation

- Each fact summary of the decision is transformed into a vector representation using the **TF-IDF** algorithm (`TfidfVectorizer` from `sklearn`).
- An alternative that is available but not used at this stage is transformer-based embeddings like **IndoBERT**.

##### 2. Data Splitting

- The dataset is split into two parts: **training data** and **testing data** with a ratio of **80:20**.
- This technique is used for training a TF-IDF + SVM classification model.

##### 3. Retrieval Model

In this stage, the system is built using **two different approaches** for retrieving the most relevant old cases to the new query:

###### a. TF-IDF + Cosine Similarity (Case-Based Reasoning Approach)

- Using the **TF-IDF vectorizer** to represent the fact summary text of all cases as numerical vectors.
- The new case query is also transformed into a vector using the same TF-IDF.
- Similarity between vectors is calculated using **cosine similarity**.
- The top-k cases with the highest similarity scores are selected as retrieval results.
- This approach is **unsupervised** and purely based on text similarity.

###### b. TF-IDF + Support Vector Machine (SVM) (Supervised Classification Approach)

- Using the **TF-IDF vectorizer** to convert fact summaries into numerical features.
- Using a **LinearSVC (SVM)** model from `sklearn` which is trained in a supervised manner with `case_id` as the target label.
- The model learns patterns from the training data and is used to predict a single case (case_id) that best matches the new query.
- This approach is **supervised learning** and focuses on classification.

Both approaches are used to complement each other in evaluating system performance in the subsequent stages.

---

##### 4. Retrieval Functions

Two `retrieve()` functions are provided, one for each approach:

- In the **TF-IDF + Cosine** approach, the `retrieve(query: str, k: int = 5)` function will:
  1. Convert the query into a TF-IDF vector.
  2. Calculate cosine similarity with all case vectors.
  3. Return the **top-k case_ids** with the highest similarity scores.

- In the **TF-IDF + SVM** approach, the `retrieve(query: str, k: int = 1)` function will:
  1. Convert the query into a TF-IDF vector.
  2. Perform classification using the SVM model.
  3. Return the **predicted case_id** from the model (top-1).

---

With this dual approach, the system can compare the effectiveness of text similarity-based retrieval methods with machine learning-based classification methods.

##### 5. Initial Testing

- **10 test queries** are prepared along with the **ground truth** (case IDs considered most relevant).
- Queries and ground truth are saved to the `data/eval/queries.json` file for evaluation purposes in the next stage.

---

#### Output

- The SVM-based classification model is saved in:  
  `03_retrieval_model.pkl`
- The TF-IDF vectorizer is saved in:  
  `03_vectorizer.pkl`
- The test query dataset is saved in:  
  `data/eval/queries.json`

Example terminal output:

[SUCCESS] Stage 3 Case Retrieval completed:

Model saved in : 03_retrieval_model.pkl

Vectorizer saved in : 03_vectorizer.pkl

10 test queries saved in : data/eval/queries.json

---

The Case Retrieval stage has been successfully implemented using a **TF-IDF + SVM** supervised classification approach. This model is now ready to be used in the prediction stage (Solution Reuse) and for performance evaluation in the next stages.

### Stage 4 - Solution Reuse
#### Objective
In this stage, the system aims to leverage solutions from past cases (court rulings) as references or basis for predicting outcomes for similar new cases.

---

#### Work Steps

1. **Solution Extraction**
   - From each retrieved past case, the system extracts the **decision text** or the **full decision content** as the solution.
   - The solution is stored in a dictionary structure in the format `{case_id: solution_text}`.

2. **Prediction Algorithm**
   Two solution prediction methods are implemented based on the approach used:

   - **TF-IDF + Cosine Similarity (CBR)**
     - The system retrieves the top-k most relevant cases based on cosine similarity against the TF-IDF vectors.
     - The solution from the first case (top-1) is considered the most representative and taken as the `predicted_solution`.

   - **TF-IDF + SVM (Supervised Classification)**
     - The system predicts one case ID (top-1) using the SVM classification model.
     - The decision text from the predicted case is taken as the `predicted_solution`.

3. **Solution Summarization**
   - To ensure readability and concise presentation, the predicted solution is shortened like an abstract (approximately the first 50 words).

4. **Manual Demo**
   - There are 10 new case queries used for testing.
   - For each query, the system runs the `predict_outcome()` function and compares the predicted solution with the context of the problem.

---

#### Main Functions

- `retrieve(query: str, k: int = 5)`: Retrieves the top-k `case_id` based on the approach used (Cosine or SVM).
- `predict_outcome(query: str) -> Tuple[str, List[int]]`: Returns the predicted solution summary and a list of `case_id`s used as references.

---

#### Output

Two prediction result files are saved in the directory:

- `data/results/predictions_cosine.csv`: Contains results predicted using **TF-IDF + Cosine Similarity**.
- `data/results/predictions_svm.csv`: Contains results predicted using **TF-IDF + SVM**.

Each prediction file includes the following columns:
- `query_id`: The query sequence number.
- `query`: The new case summary.
- `predicted_solution`: The predicted solution summary from past cases.
- `top_5_case_ids`: A list of `case_id`s from past cases used as references.

Example of file structure:

query_id,query,predicted_solution,top_5_case_ids  
1,a man kills his mother with a knife...,decision number 1323/pid.b/2024/pn mdn..., [6, 30, 1, 41, 20]  
2,stabbing due to infidelity...,decision number 1100/pid.b/2024/pn mdn..., [3, 15, 5, 42, 47]  
...

---

#### Notes
With the two approaches (unsupervised and supervised), the system’s performance can be compared in the next evaluation stage to determine which model is most effective in utilizing solutions from past cases.


### Stage 5 – Model Evaluation
#### Objective
This stage aims to measure and analyze the performance of the system in performing retrieval and predicting solutions for new cases based on past cases.

---

#### Work Steps

##### 1. Retrieval Evaluation
Evaluation is carried out on the retrieval results using two approaches:
- **TF-IDF + Cosine Similarity (unsupervised/CBR)**
- **TF-IDF + SVM (supervised classification)**

The metrics used are:
- **Accuracy**
- **Precision**
- **Recall**
- **F1-score**

Evaluation is performed by comparing the `top_k` predicted results with the ground-truth `case_id` from 10 test queries.

##### 2. Visualization & Report
- A comparison table of metrics between models is displayed as a bar chart.
- **Error analysis** for failed cases is also conducted and stored in a `.json` format.

---

#### Implementation

The main evaluation functions include:
- `eval_retrieval()`: Evaluates the cosine similarity-based approach (top-k match).
- `eval_prediction()`: Evaluates the top-1 prediction from the SVM model.
- `save_errors()`: Saves the list of queries that failed to predict correctly.

---

#### Output

Here is a summary of the model evaluation results:

##### 1. Evaluation Result Files

- **Retrieval (Cosine Similarity)**  
  Saved at: `data/eval/retrieval_metrics.csv`
  
model,accuracy,precision,recall,f1_score  
TF-IDF + Cosine,0.9,1.0,0.9,0.9473684210526315

- **Prediction (SVM)**  
Saved at: `data/eval/prediction_metrics.csv`

model,accuracy,precision,recall,f1_score  
TF-IDF + SVM,0.6,1.0,0.6,0.75

##### 2. Visualization

A performance comparison bar chart between models is saved at:  
`data/eval/performance_comparison.png`

##### 3. Error Analysis

- **Errors in TF-IDF + Cosine:**
  Saved at: `data/eval/error_cases_cosine.json`
  ```json
  [
    {
      "query_id": 7,
      "query": "the defendant killed the victim, the homeowner, while committing a burglary at his house",
      "predicted": [5, 20, 1, 43, 18],
      "ground_truth": [7]
    }
  ]

## License & Credits

This project is developed as part of an academic assignment. All code, documentation, and resources are for educational purposes and are intended to be shared on GitHub for assignment submission.

### License

This repository is licensed under the [MIT License](LICENSE), allowing for personal and educational use. Please do not use it for commercial purposes without proper authorization.

### Credits

- **Students**: [202210370311427_Mokh. Brillian Dwi Ariestianto], [202210370311445_M. Razzan Carveyna Hibrizi]  
  Developed as part of the [University Muhammadiyah Malang/Informatics] course project.
  
- **Professor/Instructor**: [Ir. Galih Wasis Wicaksono, S.kom. M.Cs.]  
  For guidance and support throughout the project.

- **Libraries Used**:
  - `pdfminer` - for PDF to text conversion
  - `scikit-learn` - for machine learning algorithms
  - `transformers` - for BERT-based retrieval
  - `pandas`, `numpy`, `matplotlib` - for data processing and visualization

- **Special Thanks**:  
  To all the contributors and open-source developers whose libraries have been used to make this project possible.

Feel free to fork, clone, and modify the project for educational use, and please credit this project appropriately if you choose to reuse or share the code.