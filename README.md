## Assignment and Analysis Plan

**Goal:** Efficiently allocate supply so that riders can always get a ride and **drivers have stable earnings**.

**Assignment Objectives:**

- **Explore the data** and suggest a **solution to guide drivers towards areas** with higher expected demand at given times and locations.
- **Build and document a baseline model** for your solution.
- **Describe** how you would **design** and **deploy** such a model.
- **Communicate** how to present model recommendations to drivers.

**Data Description:**

- **start_time**: Time when the order was made.
- **start_lat**: Latitude of the order's pick-up point.
- **start_lng**: Longitude of the order's pick-up point.
- **end_lat**: Latitude of the order's destination point.
- **end_lng**: Longitude of the order's destination point.
- **ride_value**: Monetary value of this particular ride.

**Initial ideas:**

- Create a hexagonal ([bestagonal](https://www.youtube.com/watch?v=thOifuHs6eY)) heatmap with 15-minute forecasts. Drivers would see a heatmap with darker areas indicating higher expected demand.
- Predict demand in 15-minute intervals using time series models (ARMA, SARIMA) or machine learning models like XGBoost or LSTM.
- **Defining Demand:**
  - Number of requests in a 15-minute time window within a hexagon.
  - Consider incorporating monetary value of rides by weighting demand with ride prices.
- **Infrastructure:**
  - Use Docker containers on AWS ECS for storage and deployment.
  - Build an API using FastAPI for model serving and communication.
- **Evaluation Metrics:**
  - Determine appropriate metrics: driver response time to requests, minimizing unaccepted rides, maximizing revenue.
  - Measure the increase in drivers' earnings, which indirectly corresponds to company earnings and customer wait times.
  - Implement the recommendation system (heatmap and notifications) for a random sample of drivers and observe them over a period.
  - Use difference-in-differences (DiD) to evaluate system performance.

---

## Analysis Plan

### Data Loading

1. **Load Data**
1. **Inspect Data**
   - Check for missing values.
   - Check for duplicates.
   - Check for outliers.
   - Identify strange observations (e.g., using maps for visual inspection):
     - Coordinates outside of Tallinn.
     - Zero or excessively large distances.
     - Abnormally high monetary values.
     - Coordinates located in water bodies.
1. **Data Cleaning**
   - Handle missing values.
   - Remove duplicates.
   - Normalize data for modeling.
1. **Feature Engineering**
   - Extract day of the week.
   - Determine part of the day (morning, afternoon, evening, night).
   - Identify workdays and holidays.
   - Calculate Haversine distance (straight-line distance for simplicity).
   - Estimate travel time (if possible).
   - Round timestamps to the nearest 15 minutes (forecast window).
   - Bin data into hexagons using the H3 library.
   - Create lag features from demand (e.g., 15 minutes ago, 1 hour ago, 1 day ago, 1 week ago).
   - Compute rolling averages over the previous week (168 periods for 15-minute intervals).
   - Other Potential Features: Weather data, order end time, order acceptance time, unfulfilled orders, drive time.


### Exploratory Analysis

1. **Visualizations**
   - Plot geographic data on a map.
   - Analyze the distribution of monetary values.

### Modeling

1. **Model Selection**
   - **XGBoost Model**: Given the need to account for multiple areas and non-linear relationships, and considering time constraints, focus on using XGBoost for its efficiency and ability to handle complex data.
   - **Alternatives**:
     - **Time Series Models**: ARMA, SARIMA may require separate models for each area.
     - **Vector Autoregression**: Can account for multiple areas but may be complex to implement.
     - **Neural Networks**: LSTM networks could capture temporal and spatial dependencies but may require different data structures and more time.
1. **Train-Test Split**
   - Since the data is time series, split data based on time (e.g., train on past data, test on future data).
1. **Cross-Validation**
   - Implement a time series cross-validation scheme by splitting data into sequential time windows.
   - Use each window for testing while training on all previous data.

### Model Evaluation

1. **Performance Metrics**
   - Evaluate model performance on the test set using metrics like Mean Squared Error (MSE).
1. **Baseline Models**
   - **Previous Period Predictor**: Use demand from the previous 15-minute interval as the prediction.
   - **Moving Average Predictor**: Use the moving average of the previous four periods (1 hour) as the prediction.
   - **Seasonal Predictor**: Use demand from the same time in the previous week as the prediction.
1. **XGBoost Model Evaluation**
   - Compare the XGBoost model's performance against the baseline models.

### Model Deployment

- **API Development**: Build an API using FastAPI for model serving.
- **Containerization**: Use Docker containers on AWS ECS for deployment.
- **Data Storage**: Connect to a database for data storage and retrieval.
- **Considerations**:
  - Understand the cost of serving the model, might be too high compared to value added.
  - Think about changes in forecast windows (15 min is quite frequent) or data granularity.

### Model Communication

- **Visualization**: Display demand forecasts on a map within the driver app, using color-coded heatmaps to indicate areas of higher expected demand.
- **Notifications**: Send notification to drivers when demand is expected to pick up in specific areas.

### Experiment Design

- **Randomized Controlled Trial (RCT)**
  - **Treatment Group**: Drivers who receive the recommendation system (heatmap and notifications).
  - **Control Group**: Drivers who do not receive recommendations.
- **Evaluation Methodology**
  - Use a **Difference-in-Differences (DiD)** approach to compare the impact of the recommendation system.
  - Use pre- and post-implementation data to control for driver-specific characteristics, fixed effects, and other time-invariant factors.

**Difference-in-Differences Model:**

$$
\text{Outcome}_{it} = \beta_0 + \beta_1 \text{Post}_t + \beta_2 \text{Treatment}_i + \beta_3 (\text{Post}_t \times \text{Treatment}_i) + \gamma X_{it} + \alpha_i + \lambda_t + \varepsilon_{it}
$$

Where:

- $\text{Outcome}_{it}$: Outcome variable (e.g., earnings) for driver $i$ at time $t$.
- $\text{Post}_t$: Indicator variable (1 if after treatment, 0 if before).
- $\text{Treatment}_i$: Indicator variable (1 if driver is in the treatment group, 0 if control).
- $\text{Post}_t \times \text{Treatment}_i$: Interaction term capturing the treatment effect.
- $X_{it}$: Vector of control variables.
- $\alpha_i$: Driver fixed effects.
- $\lambda_t$: Time fixed effects.
- $\varepsilon_{it}$: Error term.

- **$\beta_3$** (Coefficient on the interaction term):
  - Estimates the **Average Treatment Effect (ATE)** of the intervention.
  - Represents the difference in changes over time between the treatment and control groups. Positive $\beta_3$ would indicate treatment had higher income compared to control after the intervention.
