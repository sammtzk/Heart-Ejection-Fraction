# **Left Ventricle Ejection Fraction Prediction Using Bayesian ML**

***NOTICE:*** Neither these models, nor the EchoNet-Dynamic dataset, have been reviewed or approved by the Food and Drug Administration. The results, models, and proposals shared here are for non-clinical, **Research Use Only**. In no event shall data or predictions generated through the use of this work be used or relied upon in the diagnosis or provision of patient care.

---
## **Problem Statement & Motivation**

### Overview
The goal of this project is to develop a predictive model for **Ejection Fraction (EF)** estimation, comparing it against **traditional medical techniques**, which rely on **manual annotations** and clinician expertise.

### Motivation
- **Manual EF estimation is time-consuming**: Conventional approaches involve expert tracings of the left ventricle across multiple frames in an echocardiogram video.
- **High inter-observer variability**: Different clinicians may provide slightly different EF values, leading to inconsistencies.
- **Automating EF prediction can enhance efficiency and accessibility**: A deep learning-based approach could assist medical professionals by providing **rapid, consistent, and reproducible** EF predictions.

### Challenges
- **Complexity of cardiac dynamics**: The heart's motion is nonlinear, requiring models that can capture **both spatial and temporal features**.
- **Data standardization**: Differences in video formats, resolutions, and acquisition protocols can introduce variability.
- **Need for robust feature extraction**: Effective EF prediction requires detecting key cardiac phases (**end-diastole and end-systole**) and accurately segmenting the **left ventricle**.

By leveraging machine learning, we aim to **streamline EF estimation**, reduce manual workload, and potentially enable real-time cardiac assessment in clinical settings.

---
## **Data Set**
![frame1](./attachment/frame_0114.png) ![frame2](./attachment/frame_0156.png)
![frame3](./attachment/frame_0176.png) ![frame4](./attachment/frame_0194.png)

***DATA ACCESS DISCLOSURE:*** Access to the data used for this research is granted under the Stanford University School of Medicine EchoNet-Dynamic Dataset Research Use Agreement: [Accessing Dataset](https://echonet.github.io/dynamic/index.html#access).

Our **Cardio-Health Predictive Model** utilizes the **EchoNet-Dynamic dataset**, a publicly available compilation of over **10,000 echocardiogram videos** collected between **2016 and 2018 at Stanford University Hospital**. Each video is accompanied by **volume tracings** and **clinical measurements**, including **ejection fraction (EF)** - a measure estimating the percentage of blood pumped out of the left ventricle during a heart contraction.  

Experts at a **Stanford lab** used the videos, volume tracings, and EF measurements to design a **deep learning model** that:
- **Accurately outlines the left ventricle** in each frame  
- **Predicts Ejection Fraction (EF) values**  
- **Classifies heart failure** based on the predicted EF values  

### **Understanding Ejection Fraction (EF)**  
Ejection Fraction values can be categorized as follows:  

- **50-70% (Normal EF)**  
  - A healthy heart that pumps an adequate amount of blood out of the left ventricle per beat  
- **Below 50% (Reduced EF)**  
  - Indicates impaired pumping capability, which can lead to **mild to severe heart failure symptoms**, such as:
    - Shortness of breath  
    - Fluid buildup  
- **Above 70% (Potential Hypertrophic Cardiomyopathy)**  
  - A condition where the heart **vigorously pumps out blood** but may still face issues with refilling  

### **Ejection Fraction Formula**  
EF is calculated based on two key **cardiac cycle phases**:  
- **End-Diastolic Volume (EDV)** – Maximum ventricular volume (when the ventricle is full)  
- **End-Systolic Volume (ESV)** – Minimum ventricular volume (after contraction)  

EF = (EDV - ESV) / EDV X 100


### **Cardiac Cycle Breakdown**  

#### **1. Diastole (EDV) - Filling Phase**  
- The left **ventricle is relaxed**  
- Blood **flows in** from the **left atrium**  

#### **2. Isovolumetric Contraction**  
- The **ventricle contracts**, but **valves remain closed**  

#### **3. Systole (ESV) - Ejection Phase**  
- **Valves open**, allowing blood to be ejected from the **left ventricle into the aorta**  

#### **4. Isovolumetric Relaxation**  
- The **ventricle relaxes** before it starts **filling up again**  

<img src="https://www.cardofmich.com/wp-content/uploads/2019/10/Card-of-Mich-_-Blog-Anatomy-of-the-Heart-598167278.jpg" alt="Heart Structure" width="400" />
---

This dataset provides the foundation for **training deep learning models** that can accurately analyze heart function, predict ejection fraction, and detect potential heart failure conditions.

Our goal is to create a pipeline that predicts EF using primarily the collection of video echocardiograms provided in the EchoNet-Dyanmic dataset.

---
## **Model Pipeline**
![Flow Chart](./attachment/flowchart.png)

### **Video Standardization and Preprocessing**

In the effort to train a proof-of-concept EF prediction model from echocardiogram videos, our pipeline necessitates first constructing a subset of the videos which have identical descriptive metadata (resolution, FPS, number of frames). This is to ensure uniformity in the inputs to the ResNet-18 for transfer learning. We select the subset of videos which have the following attributes: FrameHeight: 112, FrameWidth: 112, FPS: 50, NumberOfFrames: 201. This produces a subset of 208 videos, but we are cognizant that future development of this pipeline should include broader video standardization processes to increase the size of training, testing, and validation splits.

As a preprocessing step, video frames need to be extracted before the ResNet-18 feature extraction. We use OpenCV to perform grayscale transformations of the video color data as well as basic frame extraction. The [1. Data Sampling and Frame Extraction](1_Data_Sampling_and_Frame_Extraction.ipynb) notebook saves extracted frame data using Pickle serialization to increase data loading efficiency within Python environments. Some sample code is included to demonstrate how video frames can be saved in other formats.

### **Feature Extraction with CNN**

Inputting the raw pixel data from the videos to the Gaussian HMM can lead to computational inefficiency because of the large number of pixels that are extracted per frame, which can lead to the HMM struggling to train. Besides that, the HMM will be trained with irrelevant features from the image. To mitigate this issue, we use a pre-trained CNN called ResNet18 to extract the most important features from the pixel data in a lower dimensional space. We use ResNet18 for a few reasons:

- Pretrained on millions of diverse images, where the general features should transfer well to the medical domain
- Lightweight compared to other pretrained models, balancing between computational efficiency and performance

We will be taking the final layer before the classification layer of ResNet18 to be inputted into the Gaussian HMM for training.

![image](https://github.com/user-attachments/assets/7855e80d-1056-499a-b749-4f7364c5e434)

### **Gaussian HMM Modeling and Regression**

We use a Gaussian HMM to predict the states because our data input is continuous and we need to model complex, continuous state spaces. For the amount of hidden states, we chose 4 hidden states based on our research on different states of the heart, but multiple hidden state numbers will be considered. The output of the Gaussian HMM will be a prediction of the hidden state per frame. Then we will get the distribution of the states per video and use that as the input for our regression model.

The output of the Gaussian HMM model will then be inputted in our regression model as our independent variables (X), with the EF values as our dependent variable (y). Multiple regression models are considered, such as linear regression, ridge regression and ensemble methods (random forest, gradient boosting, XGboost) and compared with each other to find the model with the best performance.

---
## **Result / Summary**

Using a hidden state number of 4, the results of the model created are not so promising, with a high test RMSE and MAPE.

<img width="448" alt="image" src="https://github.com/user-attachments/assets/4ecc42df-94fe-40eb-ada8-b892d6dbe581" />
Linear Regression

<img width="463" alt="image" src="https://github.com/user-attachments/assets/d2a44f96-1285-4430-bb0a-ef8d01a9f9ed" />
Random Forest

<img width="449" alt="image" src="https://github.com/user-attachments/assets/a61f6d24-7472-42fb-8cd3-bea11815cf62" />
Gradient Boosting

<img width="454" alt="image" src="https://github.com/user-attachments/assets/42af8096-2afc-4cc0-94fb-d0d5c5b25111" />
XGBoost

---
## **Next Steps / Future Improvements**
While our current model provides an approach to estimating ejection fraction (EF) using machine learning techniques, there are several areas for improvement and future directions.

## **1. Improving Feature Extraction**
The choice of **ResNet18** for feature extraction prioritized texture-based patterns such as **edges, curves, and gradients**. However, this may not fully capture the **temporal dynamics** of the left ventricle’s motion. To improve this:  

### **Alternative Deep Learning Models:**
- **Echonet**: A deep learning model specifically designed for echocardiography, trained on large-scale EF datasets.
- **Other 3D CNN**: There may be some other 3D convolutional model that could capture volumetric changes more effectively.
    - SlowFast Network
    - VoxResNet
- **TimeSformer**: A transformer-based architecture that models spatial and temporal dependencies better than CNNs.

## **2. Refining State Modeling (Beyond GMM-HMM)**
Our current pipeline applies **Gaussian Mixture Models (GMM) + Hidden Markov Models (HMM)** to model state transitions. However, several enhancements could be explored:

### **Alternative State Modeling Techniques:**
- **Deep Hidden Markov Models (DeepHMM)**: Uses neural networks to model latent state transitions more effectively.

### **Adaptive State Learning**
- Instead of **predefining the number of states**, use a **data-driven approach** to learn the **optimal number of EF states dynamically**.
- Why Avoid Predefining States?
    - Cardiac motion is complex and varies across patients, meaning that a fixed number of states may oversimplify or overcomplicate the EF estimation process.
    - Different patients have different ventricular motion characteristics.
    - Due to time-dependent changes in EF, using a fixed number of states may fail to capture gradual changes in EF dynamics.
- **Recurrent Neural Networks (RNNs) or LSTMs**: Can learn **temporal dependencies** in EF progression without requiring discrete states.
- **Transformer-based Sequence Models**: Could improve performance by better handling long-term dependencies.




