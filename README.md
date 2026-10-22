# Data-Analytics-Rain-Prediction

---
Title: "Predicting Next-Day Rain with Classification Models in KNIME: Evaluation Pitfalls and Target Leakage"
Author: "Brunda Varshini Katapalli"
---

# Title

**Predicting Next-Day Rain with Classification Models in KNIME: Evaluation Pitfalls and Target Leakage**

Brunda Varshini Katapalli

# Abstract

This study builds binary classifiers in KNIME to predict whether it will rain tomorrow (`RainTomorrow`) from daily weather observations. The training data contained 51,199 rows (51,187 after duplicate removal) with a class imbalance of roughly one "Rain" day to three "No Rain" days. Five model families were built: Decision Tree, k-Nearest Neighbours (k-NN), Random Forest, Support Vector Machine (SVM) and a Neural Network. Data were partitioned 70/30 with stratified sampling, SMOTE was tested for the imbalance, and parameters were tuned with brute-force search, with 10-fold cross-validation used as a comparison. Three models reported 100% hold-out accuracy, and a fourth (Random Forest) reported 79.28%. However, the same k-NN model scored 0.51542 on the Kaggle public leaderboard, which is close to chance level. This gap, together with an ROC curve that sits on the random-classifier diagonal, shows that the 100% figures are not genuine performance. Inspection of the workflow showed that the target itself was passed to the models as input features, a problem known as target leakage. The paper documents the method, reports the results honestly, and sets out how the evaluation should be corrected.

# Introduction

Rainfall prediction is a standard supervised-learning problem with practical value for agriculture, event planning and water management. The task here is binary classification: given weather observations for a day, predict whether it rains on the following day (`RainTomorrow` = 1) or not (`RainTomorrow` = 0). Models are judged on accuracy, precision, recall, F1-score and the area under the ROC curve (AUC).

The data were supplied as a CSV file of daily observations with 21 attributes plus the class: `Location`, `MinTemp`, `MaxTemp`, `Rainfall`, `Evaporation`, `Sunshine`, `WindGustDir`, `WindGustSpeed`, `WindDir9am`, `WindDir3pm`, `WindSpeed9am`, `WindSpeed3pm`, `Humidity9am`, `Humidity3pm`, `Pressure9am`, `Pressure3pm`, `Cloud9am`, `Cloud3pm`, `Temp9am`, `Temp3pm` and `RainToday`. A second file of unlabelled rows was used for final scoring on Kaggle. The raw data are not redistributed with this paper.

The objectives were to explore the data, build and tune several classifiers, compare them with sound evaluation, and select the model that is most likely to generalise. A secondary outcome of the work is a case study of how a very high validation score can hide a methodological error.

# Methodology

All work was done in KNIME Analytics Platform (Berthold et al., 2008).

## Data preparation and exploration

The workflow applied the following steps in order:

1. **Type assignment.** The Column Auto Type Cast node assigned a data type to each column.
2. **Class conversion for exploration.** A Number to String node converted `RainTomorrow` from integer to string so the Statistics node could show its distribution. The histogram showed the classes are imbalanced, with far more "No Rain" days than "Rain" days. This motivated stratified sampling and SMOTE.
3. **Duplicate removal.** The Duplicate Row Filter removed 12 identical rows (51,199 to 51,187).
4. **Missing values.** The Missing Value Column Filter identified columns with missing data and handled them before modelling.
5. **Encoding.** Categorical attributes were converted to numeric form (String to Number, and One to Many for the models that need it).
6. **Scaling.** A Normalizer rescaled numeric features so that scale-sensitive models (k-NN, SVM, neural network) weight them evenly.

A rank-correlation matrix was computed to see how attributes relate to each other and to the class. `MaxTemp`, `WindGustDir`, `WindDir9am`, `WindDir3pm`, `Pressure9am`, `Pressure3pm`, `Temp9am` and `Temp3pm` showed the weakest association with `RainTomorrow`.

## Handling imbalance and partitioning

- **Partitioning.** Labelled data were split 70% training and 30% validation using stratified sampling on `RainTomorrow`, so both partitions keep the original class ratio.
- **SMOTE.** The Synthetic Minority Over-sampling Technique (Chawla et al., 2002) was used to oversample the "Rain" class and compare each model with and without it.
- **Cross-validation.** 10-fold stratified cross-validation was run for comparison. It uses all labelled rows for both training and testing across folds, which is less dependent on a single split.

## Classifiers and parameters

| Classifier | Key settings and rationale |
|---|---|
| Decision Tree | Gain ratio as the quality measure, which avoids favouring attributes with many distinct values. MDL pruning and reduced-error pruning to limit overfitting. Minimum 2 records per node. |
| k-NN | k = 5, a common balance between bias (large k oversmooths) and variance (small k fits noise). Inputs normalised. |
| Random Forest | Information gain ratio as the split criterion; it gave better results than information gain or the Gini index in trials. |
| SVM | Polynomial kernel, chosen over HyperTangent and RBF after comparison. Power, Bias and Gamma tuned with a parameter-optimisation loop using brute-force search (Power 1 to 2 in steps of 0.2, Bias 1 to 2 in steps of 0.1, Gamma 1 to 2 in steps of 0.2). |
| Neural Network | Multilayer perceptron with a parameter-optimisation loop. |
| 10-fold cross-validation | Stratified sampling on `RainTomorrow`, with k-NN as the inner model. |

Parameter optimisation strategies tried were brute force, hill climbing, random search and Bayesian optimisation (TPE). Brute force was retained for the SVM because its search space was small enough to enumerate fully.

# Results

## Reported hold-out performance

| Model | Reported accuracy | Notes |
|---|---|---|
| Decision Tree (with SMOTE) | 100% | Zero misclassifications on the validation partition |
| k-NN (with and without SMOTE) | 100% | Same result with and without SMOTE |
| SVM (polynomial kernel) | 100% | 15,357 validation rows classified correctly (11,916 No Rain, 3,441 Rain); Cohen's kappa 1; ROC curve on the diagonal |
| Random Forest | 79.28% | 20.72% error. Predicted probability for the sample row: 0.7697 No Rain, 0.2303 Rain |
| Neural Network | Not reported | Workflow built, results not recorded in the write-up |

## External result

The selected k-NN model (k = 5, with SMOTE) was applied to the unlabelled file and submitted to Kaggle. The public score was **0.51542**. This metric is AUC, where 0.5 is a random classifier and 1.0 is perfect.

## Why the 100% results should not be trusted

Four observations point to the same conclusion:

1. **Three unrelated model families reached exactly 100%.** A decision tree, k-NN and an SVM would not normally all separate next-day rain perfectly from weather variables, especially since Random Forest on the same data reached 79.28%.
2. **The external score contradicts the internal score.** A model that is perfect on validation data but scores 0.51542 on unseen data is behaving as though it had access to the answer during validation and not during prediction.
3. **The SVM ROC curve is the random-classifier diagonal**, which cannot be true of a perfect classifier.
4. **The SVM confusion matrix shows the class labels as -0.537 and 1.860**, not 0 and 1. These are standardised values of `RainTomorrow`, which indicates that a numeric copy of the target had been scaled and was travelling through the pipeline as data.

## Cause: target leakage in the workflow

Inspection of the saved KNIME workflow confirmed the cause (target leakage; Kaufman et al., 2012). In every model branch, a One to Many node was applied to `RainTomorrow` itself with the option to remove the source column switched off. The node therefore added one indicator column per class value (an exact 0/1 copy of the answer) to the table, and the original `RainTomorrow` column was still used as the class. The Decision Tree, k-NN and SVM learners were all given those indicator columns as inputs and could read the label directly, which explains the perfect validation scores. The unlabelled Kaggle file has no `RainTomorrow` column, so the same columns do not exist there and the model had nothing to rely on, which explains the score near 0.5.

Further sources of optimism in the pipeline, which should be fixed regardless:

- **Preprocessing before partitioning.** Normalisation and encoding were fitted on all rows before the split. Statistics from the validation rows therefore influenced training data.
- **SMOTE placement.** SMOTE should be applied to the training partition only, never before the split or to validation data. Oversampling before partitioning places near-copies of the same observation on both sides of the split.
- **Missing target documentation.** Neural Network results were not recorded.

# Conclusion

The workflow covers the full classification process in KNIME: exploration, cleaning, encoding, scaling, stratified partitioning, SMOTE, parameter optimisation and cross-validation across five model families. The only result that appears plausible is the Random Forest's 79.28% accuracy, which is also barely above the roughly 78% (11,916 of 15,357) that a classifier gets by always predicting "No Rain" on the validation class balance, so it should not be read as strong skill.

The headline 100% accuracy was a warning sign and not a success. The Kaggle AUC of 0.51542 shows that the selected k-NN model has essentially no predictive power on unseen data, and the workflow inspection confirms target leakage through the One to Many nodes. The k-NN model was selected on the basis of the flawed validation result, so that selection is not supported.

A corrected study would:

1. Exclude `RainTomorrow` from the One to Many node (or enable removal of the source column), and check each Learner node's input columns explicitly.
2. Keep the class column out of the Normalizer.
3. Partition first, then fit normalisation and encoding on the training partition only and apply them to validation data.
4. Apply SMOTE to the training partition only.
5. Report precision, recall, F1 and AUC alongside accuracy, and treat any score near 100% as a prompt to investigate before reporting.
6. Re-run model selection and resubmit to Kaggle to confirm the validation and external scores agree.

# References

Berthold, M. R., Cebron, N., Dill, F., Gabriel, T. R., Kötter, T., Meinl, T., Ohl, P., Sieb, C., Thiel, K., & Wiswedel, B. (2008). KNIME: The Konstanz Information Miner. In *Data Analysis, Machine Learning and Applications* (pp. 319-326). Springer.

Breiman, L. (2001). Random forests. *Machine Learning, 45*(1), 5-32.

Chawla, N. V., Bowyer, K. W., Hall, L. O., & Kegelmeyer, W. P. (2002). SMOTE: Synthetic minority over-sampling technique. *Journal of Artificial Intelligence Research, 16*, 321-357.

Cortes, C., & Vapnik, V. (1995). Support-vector networks. *Machine Learning, 20*(3), 273-297.

Cover, T., & Hart, P. (1967). Nearest neighbor pattern classification. *IEEE Transactions on Information Theory, 13*(1), 21-27.

Kaufman, S., Rosset, S., Perlich, C., & Stitelman, O. (2012). Leakage in data mining: Formulation, detection, and avoidance. *ACM Transactions on Knowledge Discovery from Data, 6*(4), 1-21.
