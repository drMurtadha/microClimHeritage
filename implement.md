# Predicting Visitor Flow Based on Microclimate Data

## 1. Data Collection and Preprocessing
- **Microclimate Data:** Gather historical data on temperature, humidity, precipitation, wind speed, etc.
- **Visitor Data:** Collect visitor counts, timestamps, and other relevant information.
- **Data Cleaning:** Handle missing values, outliers, and normalize the data using techniques like Z-score normalization.

## 2. Feature Engineering
- **Temporal Features:** Extract features like time of day, day of the week, and seasonality.
- **Weather Features:** Use weather-related features derived from microclimate data.
- **Interaction Features:** Create features that capture interactions between weather and visitor data.

## 3. Model Training

### Random Forest Training:
```python
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestRegressor
from sklearn.metrics import mean_absolute_error, mean_squared_error
import numpy as np

# Data Preparation
X_train, X_valid, y_train, y_valid = train_test_split(X, y, test_size=0.2, random_state=42)

# Model Initialization
rf_model = RandomForestRegressor(n_estimators=100, max_depth=None, random_state=42)

# Model Training
rf_model.fit(X_train, y_train)

# Model Prediction
rf_preds = rf_model.predict(X_valid)

# Evaluation
rf_mae = mean_absolute_error(y_valid, rf_preds)
rf_rmse = np.sqrt(mean_squared_error(y_valid, rf_preds))

print("RF MAE:", rf_mae)
print("RF RMSE:", rf_rmse)
```

### LSTM Training:
```python
import numpy as np
import tensorflow as tf
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import LSTM, Dense, Dropout
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_absolute_error, mean_squared_error

def create_sequences(data, sequence_length):
    xs, ys = [], []
    for i in range(len(data) - sequence_length):
        x = data[i:i+sequence_length]
        y = data[i+sequence_length]
        xs.append(x)
        ys.append(y)
    return np.array(xs), np.array(ys)

sequence_length = 10
X_seq, y_seq = create_sequences(np.hstack((X, y.reshape(-1, 1))), sequence_length)
X_train_seq, X_valid_seq, y_train_seq, y_valid_seq = train_test_split(X_seq, y_seq, test_size=0.2, random_state=42)

# Model Initialization
lstm_model = Sequential([
    LSTM(50, return_sequences=True, input_shape=(X_train_seq.shape[1], X_train_seq.shape[2])),
    Dropout(0.2),
    LSTM(50),
    Dropout(0.2),
    Dense(1)
])

lstm_model.compile(optimizer='adam', loss='mean_squared_error')

# Model Training
lstm_model.fit(X_train_seq, y_train_seq, epochs=50, batch_size=32, validation_data=(X_valid_seq, y_valid_seq))

# Model Prediction
lstm_preds = lstm_model.predict(X_valid_seq)
lstm_preds = lstm_preds.flatten()  # Flatten the predictions to match the shape of y_valid_seq

# Evaluation
lstm_mae = mean_absolute_error(y_valid_seq, lstm_preds)
lstm_rmse = np.sqrt(mean_squared_error(y_valid_seq, lstm_preds))

print("LSTM MAE:", lstm_mae)
print("LSTM RMSE:", lstm_rmse)
```

## 4. Model Optimization with Grey Wolf Optimization (GWO)

### Hyperparameter Tuning:
```python
from grey_wolf_optimizer import GreyWolfOptimizer
from sklearn.ensemble import RandomForestRegressor
from sklearn.metrics import mean_absolute_error

def objective_function(params):
    # Extract hyperparameters
    n_estimators = int(params[0])
    max_depth = int(params[1])

    # Train Random Forest model
    rf_model = RandomForestRegressor(n_estimators=n_estimators, max_depth=max_depth)
    rf_model.fit(X_train, y_train)

    # Predict and calculate MAE
    predictions = rf_model.predict(X_valid)
    mae = mean_absolute_error(y_valid, predictions)

    return mae

# Define search space: [n_estimators, max_depth]
search_space = [(10, 200), (5, 50)]

# Initialize GWO
gwo = GreyWolfOptimizer(search_space, objective_function, n_iterations=50, n_wolves=10)
best_params = gwo.run()

print("Best Hyperparameters:", best_params)
```

### Feature Selection:
```python
def feature_selection_objective(features):
    # Select subset of features
    selected_features = X_train[:, features]

    # Train and evaluate model
    rf_model.fit(selected_features, y_train)
    predictions = rf_model.predict(X_valid[:, features])
    mae = mean_absolute_error(y_valid, predictions)

    return mae

# Define search space: feature indices
feature_indices = list(range(X_train.shape[1]))
search_space = [feature_indices for _ in range(n_wolves)]

# Initialize GWO for feature selection
gwo = GreyWolfOptimizer(search_space, feature_selection_objective, n_iterations=50, n_wolves=10)
best_features = gwo.run()

print("Best Feature Subset:", best_features)
```

## 5. Model Evaluation

### Cross-Validation:
```python
from sklearn.model_selection import KFold
from sklearn.metrics import mean_squared_error
import numpy as np

kf = KFold(n_splits=5)
rf_errors, lstm_errors = []

for train_index, test_index in kf.split(X):
    X_train, X_test = X[train_index], X[test_index]
    y_train, y_test = y[train_index], y[test_index]

    # Train and predict with Random Forest
    rf_model.fit(X_train, y_train)
    rf_preds = rf_model.predict(X_test)
    rf_errors.append(mean_squared_error(y_test, rf_preds))

    # Train and predict with LSTM (assuming lstm_model is a function that trains and predicts)
    lstm_preds = lstm_model(X_train, y_train, X_test)
    lstm_errors.append(mean_squared_error(y_test, lstm_preds))

print("Average RF MSE:", np.mean(rf_errors))
print("Average LSTM MSE:", np.mean(lstm_errors))
```

### Performance Metrics:
```python
from sklearn.metrics import mean_absolute_error, r2_score

rf_mae = mean_absolute_error(y_test, rf_preds)
rf_rmse = np.sqrt(mean_squared_error(y_test, rf_preds))
rf_r2 = r2_score(y_test, rf_preds)

print("RF MAE:", rf_mae)
print("RF RMSE:", rf_rmse)
print("RF R-squared:", rf_r2)
```

## 6. Model Integration and Prediction

### Ensemble Approach:
```python
# Combine Predictions using a weighted average
def ensemble_predictions(rf_preds, lstm_preds, w_rf=0.5, w_lstm=0.5):
    return w_rf * rf_preds + w_lstm * lstm_preds

# Example
combined_preds = ensemble_predictions(rf_preds, lstm_preds)
```

### Real-Time Prediction:
```python
# This example assumes you have a server or cloud service set up to handle real-time data inputs and model predictions
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/predict', methods=['POST'])
def predict():
    data = request.get_json()
    # Preprocess the data
    preprocessed_data = preprocess(data)
    
    # Model Predictions
    rf_preds = rf_model.predict(preprocessed_data)
    lstm_preds = lstm_model.predict(preprocessed_data)
    
    # Ensemble Prediction
    combined_preds = ensemble_predictions(rf_preds, lstm_preds)
    
    return jsonify({'prediction': combined_preds.tolist()})

if __name__ == '__main__':
    app.run(debug=True)
```
