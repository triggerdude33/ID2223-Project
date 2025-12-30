# **Alpine Resort Shutdown Model: Technical Specification**

## **1\. Executive Summary**

The **Shutdown Model** is a Sequence-to-Sequence (Seq2Seq) Recurrent Neural Network designed to predict the operational status of a ski resort over a variable time horizon.  
Unlike static classifiers that require fixed input windows (e.g., exactly 15 years), this model accepts a weather sequence of *any* length ($N$ years) and outputs a corresponding sequence of $N$ probabilities. This allows for flexible inference scenarios and precise temporal localization of the shutdown event.

## **2\. Core Architecture**

* **Model Type:** Many-to-Many LSTM (Long Short-Term Memory).  
* **Input Shape:** (Batch\_Size, Timesteps, Features)  
  * *Timesteps:* Variable ($1$ to $N$).  
  * *Features:* Weather metrics (Temp, Snow, Rain) \+ Resort static traits (Slope, Altitude).  
* **Output Shape:** (Batch\_Size, Timesteps, 1\)  
  * *Output:* A probability score ($0.0 \- 1.0$) for *each* year in the input sequence.

## **3\. The "Ghost Data" Training Strategy**

A critical requirement is that the model must be robust to "post-mortem" data—weather patterns that occur after a resort has theoretically closed.

### **3.1 The Cumulative Labeling Logic**

We treat the resort's status as a state that, once lost, cannot be recovered.

* **Scenario:** A resort operates for 5 years and shuts down due to climate stress. We possess 15 years of weather data for this location.  
* **Input Sequence (**$X$**):** 15 years of valid weather data.  
  * *Years 1-5:* Weather that caused the shutdown.  
  * *Years 6-15:* "Ghost Data" (Ambient weather continuing after closure).  
* **Target Sequence (**$y$**):**  
  * \[0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1\]  
  * *0 \= Alive*  
  * *1 \= Shutdown*

Why this works:  
This teaches the LSTM that once the internal "stress" accumulation crosses a threshold (at Year 5), the output should flip to 1 and stay there, regardless of whether the weather in Year 8 is good or bad.

## **4\. Model Implementation Details**

### **4.1 Layers**

1. **Input Layer:** shape=(None, n\_features)  
   * The None dimension enables the variable-length flexibility.  
2. **Masking Layer:** Masking(mask\_value=-999)  
   * Handles padding for batch training without affecting inference.  
3. **LSTM Layers:** return\_sequences=True  
   * **Crucial:** This setting ensures the model outputs a prediction at every single step, preserving the temporal dimension.  
4. **Output Layer:** TimeDistributed(Dense(1, activation='sigmoid'))  
   * Applies a binary classifier to each year independently, while maintaining the context of previous years.

### **4.2 Hyperparameters (Provisional)**

* **Loss Function:** binary\_crossentropy  
* **Optimizer:** Adam  
* **Metrics:** Accuracy, Precision (specifically on the transition point).

## **5\. Inference Workflow**

The inference process is loop-free and computationally efficient.  
Step 1: Data Ingestion  
Receive a block of projected weather data from the climate model.

* *Input:* \[Year 1 Weather, Year 2 Weather, ... Year 12 Weather\]

Step 2: Single Pass Prediction  
Feed the array into the model.

* *Output:* \[0.01, 0.02, 0.45, 0.88, 0.99, 0.99, ...\]

Step 3: Interpretation  
Scan the probability array for the first crossing of the decision threshold (e.g., $\> 0.5$).

* *If probs\[4\] \> 0.5:*  
  * **Result:** Shutdown predicted at **Year 5**.  
* *If no value \> 0.5:*  
  * **Result:** Resort remains **Operational** for the full duration of the provided forecast.

## **6\. Advantages Over Previous Approaches**

| Feature | Old Strategy (XGBoost/Static) | New Strategy (Seq2Seq LSTM) |
| :---- | :---- | :---- |
| **Input Length** | Fixed (Must be 15 years) | **Flexible** (5, 10, or 50 years) |
| **Resolution** | "Class 5" (One label per block) | **Year-by-Year** probabilities |
| **Post-Shutdown** | Required careful manual windowing | Handled naturally via state memory |
| **Causality** | Harder to enforce (can peak ahead) | **Strictly Causal** (Year 5 only sees Years 1-5) |

## **7\. Python Class Structure**

class ResortShutdownModel:  
    def \_\_init\_\_(self, n\_features):  
        self.model \= self.\_build\_architecture(n\_features)  
      
    def \_build\_architecture(self, n\_features):  
        inputs \= Input(shape=(None, n\_features))  
        x \= Masking(mask\_value=-999)(inputs)  
        x \= LSTM(64, return\_sequences=True)(x) \# Memory of past stress  
        x \= LSTM(32, return\_sequences=True)(x)  
        outputs \= TimeDistributed(Dense(1, activation='sigmoid'))(x)  
        return Model(inputs, outputs)

    def predict\_status(self, weather\_sequence):  
        \# weather\_sequence shape: (N\_Years, Features)  
        \# Returns: (N\_Years,) probability array  
        return self.model.predict(np.expand\_dims(weather\_sequence, 0))\[0\]  
