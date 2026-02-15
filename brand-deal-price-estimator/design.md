# Design Document: Social Media Agent

## Overview

The Social Media Agent is a full-stack web application that leverages machine learning and natural language processing to provide data-driven pricing recommendations for influencer-brand collaborations. The system collects creator metrics from multiple social media platforms, analyzes content quality and audience demographics, and uses a trained ML model to predict fair market pricing.

The architecture follows a microservices-inspired design with clear separation between data collection, analysis, and presentation layers. The system is designed to scale horizontally and handle growing user loads while maintaining data security and consistency.

## Architecture

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  Web UI      │  │  Mobile UI   │  │  API Clients │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    API Gateway Layer                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Authentication & Authorization                          │   │
│  │  Rate Limiting & Request Validation                      │   │
│  │  Request Routing & Load Balancing                        │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Service Layer                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ Auth Service │  │ Profile Svc  │  │ Analytics    │           │
│  │              │  │              │  │ Service      │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ NLP Service  │  │ ML Service   │  │ Pricing Svc  │           │
│  │              │  │              │  │              │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Data Layer                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ PostgreSQL   │  │ Redis Cache  │  │ File Storage │           │
│  │ (Primary DB) │  │ (Session &   │  │ (Models,     │           │
│  │              │  │  Cache)      │  │  Exports)    │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    External Integrations                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ YouTube API  │  │ TikTok API   │  │ Instagram    │           │
│  │              │  │              │  │ Graph API    │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│  ┌──────────────┐  ┌──────────────┐                             │
│  │ Twitter API  │  │ Email Service│                             │
│  │              │  │              │                             │
│  └──────────────┘  └──────────────┘                             │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow Pipeline

```
Creator Input
    ↓
Social Media API Fetch
    ↓
Data Validation & Normalization
    ↓
Engagement Analytics Calculation
    ↓
Audience Demographics Extraction
    ↓
Content Performance Analysis
    ↓
NLP Content Quality Analysis
    ↓
ML Model Feature Engineering
    ↓
Pricing Prediction
    ↓
Benchmark Comparison
    ↓
Recommendation Generation
    ↓
Dashboard Visualization
```

## Components and Interfaces

### 1. Authentication Service

**Responsibility**: User registration, login, password management, session handling

**Key Functions**:
- `register(email, password, user_type)` → User object with verification token
- `verify_email(token)` → Boolean (success/failure)
- `login(email, password)` → Session token
- `logout(session_token)` → Boolean
- `reset_password(email)` → Boolean (sends reset link)
- `update_password(token, new_password)` → Boolean

**Technology**: 
- Framework: Flask/FastAPI with JWT tokens
- Password hashing: Argon2
- Session storage: Redis with 24-hour expiration

### 2. Profile Management Service

**Responsibility**: Creator profile data, social media handle management, manual metric input

**Key Functions**:
- `create_profile(user_id, profile_data)` → Profile object
- `update_profile(user_id, profile_data)` → Profile object
- `add_social_media_handle(user_id, platform, handle)` → Boolean
- `get_profile(user_id)` → Profile object
- `validate_social_handle(platform, handle)` → Boolean

**Data Model**:
```
Profile {
  user_id: UUID
  creator_name: String
  bio: String
  category: String (e.g., "Beauty", "Tech", "Fitness")
  social_handles: {
    youtube: String,
    tiktok: String,
    instagram: String,
    twitter: String
  }
  manual_metrics: {
    followers: Integer,
    avg_views: Integer,
    engagement_rate: Float
  }
  created_at: DateTime
  updated_at: DateTime
}
```

### 3. Social Media Integration Service

**Responsibility**: Fetch data from social media APIs, handle authentication, manage rate limits

**Key Functions**:
- `authenticate_platform(user_id, platform, oauth_token)` → Boolean
- `fetch_creator_metrics(user_id, platform)` → Metrics object
- `fetch_audience_demographics(user_id, platform)` → Demographics object
- `fetch_content_samples(user_id, platform, limit=50)` → List[Content]
- `refresh_all_metrics(user_id)` → Boolean

**Supported Platforms**: YouTube, TikTok, Instagram, Twitter (future: LinkedIn, Twitch, Snapchat, Pinterest)

**Rate Limiting Strategy**:
- Implement exponential backoff for API failures
- Queue requests when rate limits are reached
- Cache results for 24 hours to minimize API calls

### 4. Analytics Service

**Responsibility**: Calculate engagement metrics, track trends, generate analytics

**Key Functions**:
- `calculate_engagement_rate(followers, interactions)` → Float
- `calculate_content_consistency(content_list)` → Float (0-100)
- `calculate_audience_quality_score(demographics, engagement)` → Float (0-100)
- `get_engagement_trends(user_id, days=90)` → List[DailyMetrics]
- `identify_top_content(user_id, limit=10)` → List[Content]

**Metrics Calculated**:
- Engagement Rate: (total_interactions / total_followers) * 100
- Content Consistency: Standard deviation of engagement across content pieces
- Audience Quality: Weighted score based on demographics alignment and engagement
- Growth Rate: Month-over-month follower growth percentage

### 5. NLP Content Analysis Service

**Responsibility**: Analyze content quality, sentiment, brand safety

**Key Functions**:
- `analyze_sentiment(text)` → SentimentScore (-1 to 1)
- `calculate_brand_safety_score(text, content_type)` → Float (0-100)
- `calculate_content_quality_score(text)` → Float (0-100)
- `analyze_content_batch(content_list)` → List[ContentAnalysis]
- `extract_topics(text)` → List[String]

**NLP Models**:
- Sentiment Analysis: Hugging Face Transformers (DistilBERT)
- Brand Safety: Custom fine-tuned model on brand safety dataset
- Content Quality: Combination of readability metrics and semantic analysis

**Brand Safety Scoring Logic**:
- Controversial topics: -20 points
- Negative sentiment: -15 points
- Profanity/offensive language: -25 points
- Professional tone: +10 points
- High engagement indicators: +15 points

### 6. ML Pricing Model Service

**Responsibility**: Train and serve ML model for pricing predictions

**Key Functions**:
- `predict_price(features)` → PricingPrediction
- `train_model(training_data)` → Model metrics
- `get_model_performance()` → ModelMetrics
- `predict_with_confidence(features)` → (price_range, confidence_score)

**Model Architecture**:
- Algorithm: Gradient Boosting (XGBoost or LightGBM)
- Input Features:
  - Engagement rate (normalized 0-100)
  - Follower count (log-scaled)
  - Content quality score (0-100)
  - Brand safety score (0-100)
  - Audience quality score (0-100)
  - Content consistency score (0-100)
  - Growth rate (normalized)
  - Category multiplier (category-specific adjustment)
  
- Output: Price range (min, recommended, max) in USD
- Confidence Score: Model's prediction uncertainty (0-100)

**Training Data**:
- Historical pricing data from brand collaborations
- Creator metrics at time of collaboration
- Actual deal outcomes and performance
- Minimum 1000 historical data points for initial training

### 7. Pricing Recommendation Engine

**Responsibility**: Generate pricing recommendations with context and adjustments

**Key Functions**:
- `generate_recommendation(user_id, campaign_params)` → Recommendation
- `adjust_for_campaign_type(base_price, campaign_type)` → Float
- `adjust_for_duration(base_price, duration_days)` → Float
- `adjust_for_deliverables(base_price, deliverables)` → Float
- `get_recommendation_history(user_id)` → List[Recommendation]

**Recommendation Object**:
```
Recommendation {
  user_id: UUID
  min_price: Float (USD)
  recommended_price: Float (USD)
  max_price: Float (USD)
  confidence_score: Float (0-100)
  key_factors: List[String]
  benchmark_percentile: Float (0-100)
  campaign_params: {
    category: String,
    duration_days: Integer,
    deliverables: List[String]
  }
  created_at: DateTime
}
```

**Adjustment Factors**:
- Campaign Type: Brand awareness (+0%), Product launch (+15%), Exclusive partnership (+25%)
- Duration: Per day rate * duration, with volume discounts for longer campaigns
- Deliverables: Base price + (number_of_posts * post_multiplier) + (video_content * video_multiplier)

### 8. Benchmark Service

**Responsibility**: Maintain benchmark data, compare against industry standards

**Key Functions**:
- `get_benchmark_data(category, audience_size_range)` → BenchmarkData
- `calculate_percentile_rank(user_id, category)` → Float (0-100)
- `compare_to_benchmark(user_id)` → ComparisonResult
- `update_benchmarks(new_data)` → Boolean

**Benchmark Data Structure**:
```
BenchmarkData {
  category: String
  audience_size_range: (min, max)
  avg_price: Float
  median_price: Float
  price_std_dev: Float
  engagement_rate_avg: Float
  content_quality_avg: Float
  sample_size: Integer
  last_updated: DateTime
}
```

### 9. Dashboard Service

**Responsibility**: Aggregate data for dashboard display, handle visualization requests

**Key Functions**:
- `get_dashboard_data(user_id)` → DashboardData
- `get_metrics_summary(user_id)` → MetricsSummary
- `get_chart_data(user_id, chart_type, time_range)` → ChartData
- `export_report(user_id, format)` → File (PDF or CSV)

**Dashboard Sections**:
- Key Metrics Summary (followers, engagement rate, content quality)
- Engagement Trends (30/60/90 day charts)
- Audience Demographics (pie charts by age, location, interests)
- Content Performance (top performing content, category breakdown)
- Pricing Recommendation (current recommendation with factors)
- Benchmark Comparison (percentile ranking, comparison chart)
- Recommendation History (past recommendations with trend)

### 10. Reporting and Export Service

**Responsibility**: Generate reports in multiple formats, manage export functionality

**Key Functions**:
- `generate_pdf_report(user_id, sections)` → PDF file
- `generate_csv_export(user_id, data_type)` → CSV file
- `create_shareable_link(report_id, expiration_hours)` → String (URL)
- `get_report_history(user_id)` → List[Report]

**Report Sections**:
- Executive Summary
- Creator Profile & Metrics
- Engagement Analytics
- Audience Demographics
- Content Performance Analysis
- Pricing Recommendation
- Benchmark Comparison
- Historical Trends

## Data Models

### User Model
```
User {
  id: UUID (primary key)
  email: String (unique)
  password_hash: String
  user_type: Enum ("creator", "brand_manager", "admin")
  first_name: String
  last_name: String
  is_verified: Boolean
  is_active: Boolean
  created_at: DateTime
  updated_at: DateTime
  last_login: DateTime
}
```

### Creator Profile Model
```
CreatorProfile {
  id: UUID (primary key)
  user_id: UUID (foreign key)
  category: String
  bio: String
  website: String
  social_handles: JSON {
    youtube: String,
    tiktok: String,
    instagram: String,
    twitter: String,
    linkedin: String,
    twitch: String
  }
  created_at: DateTime
  updated_at: DateTime
}
```

### Engagement Metrics Model
```
EngagementMetrics {
  id: UUID (primary key)
  creator_id: UUID (foreign key)
  platform: String
  followers: Integer
  avg_views: Integer
  avg_likes: Integer
  avg_comments: Integer
  avg_shares: Integer
  engagement_rate: Float
  growth_rate: Float
  recorded_at: DateTime
}
```

### Audience Demographics Model
```
AudienceDemographics {
  id: UUID (primary key)
  creator_id: UUID (foreign key)
  platform: String
  age_18_24: Float (percentage)
  age_25_34: Float (percentage)
  age_35_44: Float (percentage)
  age_45_54: Float (percentage)
  age_55_plus: Float (percentage)
  gender_male: Float (percentage)
  gender_female: Float (percentage)
  top_countries: JSON (list of countries with percentages)
  top_interests: JSON (list of interests with percentages)
  recorded_at: DateTime
}
```

### Content Performance Model
```
ContentPerformance {
  id: UUID (primary key)
  creator_id: UUID (foreign key)
  platform: String
  content_id: String (platform-specific ID)
  content_type: String ("post", "video", "story", "reel")
  title: String
  description: String
  views: Integer
  likes: Integer
  comments: Integer
  shares: Integer
  engagement_rate: Float
  published_at: DateTime
  recorded_at: DateTime
}
```

### Content Analysis Model
```
ContentAnalysis {
  id: UUID (primary key)
  content_id: UUID (foreign key)
  sentiment_score: Float (-1 to 1)
  brand_safety_score: Float (0-100)
  content_quality_score: Float (0-100)
  topics: JSON (list of extracted topics)
  readability_score: Float (0-100)
  analyzed_at: DateTime
}
```

### Pricing Recommendation Model
```
PricingRecommendation {
  id: UUID (primary key)
  creator_id: UUID (foreign key)
  min_price: Float (USD)
  recommended_price: Float (USD)
  max_price: Float (USD)
  confidence_score: Float (0-100)
  key_factors: JSON (list of factors influencing price)
  benchmark_percentile: Float (0-100)
  campaign_category: String
  campaign_duration_days: Integer
  deliverables: JSON (list of deliverables)
  created_at: DateTime
}
```

### Benchmark Data Model
```
BenchmarkData {
  id: UUID (primary key)
  category: String
  audience_size_min: Integer
  audience_size_max: Integer
  avg_price: Float
  median_price: Float
  price_std_dev: Float
  engagement_rate_avg: Float
  content_quality_avg: Float
  sample_size: Integer
  last_updated: DateTime
}
```

## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Property 1: Authentication Round Trip

**For any** valid user credentials (email and password), after registration and email verification, the user should be able to log in successfully and receive a valid session token.

**Validates: Requirements 1.1, 1.2, 1.3, 1.4**

### Property 2: Profile Data Persistence

**For any** creator profile with valid data, after updating the profile, retrieving the profile should return the exact same data that was submitted.

**Validates: Requirements 2.1, 2.2, 2.3**

### Property 3: Engagement Rate Calculation Correctness

**For any** set of engagement metrics (followers, interactions), the calculated engagement rate should equal (total_interactions / total_followers) * 100, and should be between 0 and 100.

**Validates: Requirements 3.4**

### Property 4: Engagement Data Immutability on Fetch

**For any** creator with linked social media accounts, fetching engagement data multiple times within the same hour should return identical metrics (assuming no actual social media changes).

**Validates: Requirements 3.1, 3.2**

### Property 5: Demographic Percentage Validation

**For any** audience demographic data, the sum of all percentage categories (age groups, gender, interests) should equal 100% for each category type.

**Validates: Requirements 4.5**

### Property 6: Content Performance Consistency

**For any** content piece, the engagement rate calculated from individual metrics (likes, comments, shares) should be consistent with the stored engagement_rate field.

**Validates: Requirements 5.2, 5.3**

### Property 7: NLP Analysis Determinism

**For any** content sample, running NLP analysis multiple times on the same content should produce identical sentiment, brand safety, and quality scores.

**Validates: Requirements 6.1, 6.2**

### Property 8: ML Model Output Validity

**For any** valid set of input features, the ML model should produce a price range where min_price ≤ recommended_price ≤ max_price, and all prices should be positive numbers.

**Validates: Requirements 7.2, 7.3**

### Property 9: Confidence Score Bounds

**For any** pricing prediction, the confidence score should be between 0 and 100, and should be inversely correlated with the price range width (wider ranges indicate lower confidence).

**Validates: Requirements 7.4**

### Property 10: Recommendation Factor Completeness

**For any** pricing recommendation, the key_factors list should include at least 3 factors that influenced the pricing, and each factor should be a non-empty string.

**Validates: Requirements 8.3**

### Property 11: Campaign Duration Adjustment Monotonicity

**For any** base price and campaign duration, increasing the duration should result in a higher total price (assuming linear or volume-discount pricing model).

**Validates: Requirements 8.5**

### Property 12: Deliverables Price Adjustment

**For any** base price and set of deliverables, adding more deliverables should result in a higher adjusted price.

**Validates: Requirements 8.6**

### Property 13: Benchmark Percentile Validity

**For any** creator, the benchmark percentile rank should be between 0 and 100, and should be consistent with the creator's position relative to other creators in the same category.

**Validates: Requirements 10.2, 10.5**

### Property 14: Data Encryption at Rest

**For any** sensitive user data (passwords, API tokens), the stored data should be encrypted and should not be readable in plaintext in the database.

**Validates: Requirements 11.1, 15.3**

### Property 15: Session Token Expiration

**For any** user session, after the session expiration time (24 hours), the session token should no longer be valid for authentication.

**Validates: Requirements 15.4, 15.5**

### Property 16: Audit Log Completeness

**For any** user action (login, profile update, recommendation generation), an audit log entry should be created with timestamp, user ID, action type, and result.

**Validates: Requirements 17.1, 17.2**

### Property 17: Export Data Consistency

**For any** exported report (PDF or CSV), the data in the export should match the data displayed in the dashboard at the time of export.

**Validates: Requirements 16.1, 16.2, 16.3**

### Property 18: API Rate Limit Handling

**For any** social media API call that exceeds rate limits, the system should queue the request and retry after the rate limit window expires, eventually succeeding without data loss.

**Validates: Requirements 12.4**

### Property 19: Error Message Specificity

**For any** validation error, the error message should clearly indicate which field is invalid and what the acceptable range or format is.

**Validates: Requirements 13.1, 13.2**

### Property 20: Database Transaction Atomicity

**For any** multi-step database operation (e.g., creating user and profile), either all steps succeed and are committed, or all steps are rolled back with no partial data.

**Validates: Requirements 11.2, 13.6**

## Error Handling

### Authentication Errors
- Invalid credentials: Return 401 Unauthorized with message "Invalid email or password"
- Account not verified: Return 403 Forbidden with message "Please verify your email before logging in"
- Session expired: Return 401 Unauthorized with message "Your session has expired. Please log in again"
- Account locked: Return 403 Forbidden with message "Your account has been locked due to suspicious activity"

### Validation Errors
- Missing required fields: Return 400 Bad Request with list of missing fields
- Invalid data format: Return 400 Bad Request with specific field and expected format
- Out of range values: Return 400 Bad Request with acceptable range
- Duplicate email: Return 409 Conflict with message "Email already registered"

### API Integration Errors
- API authentication failed: Log error, display user message "Unable to connect to social media account"
- Rate limit exceeded: Queue request, retry with exponential backoff, notify user if persistent
- Invalid social media handle: Return 400 Bad Request with message "Social media handle not found"
- API timeout: Retry up to 3 times, then return 504 Gateway Timeout

### Data Processing Errors
- NLP analysis failure: Log error, use fallback scores, notify user
- ML model prediction failure: Return 500 Internal Server Error, log error for investigation
- Database connection failure: Return 503 Service Unavailable, implement connection retry logic
- File export failure: Return 500 Internal Server Error with message "Unable to generate report"

### Recovery Strategies
- Implement circuit breaker pattern for external API calls
- Use exponential backoff for retries (1s, 2s, 4s, 8s, 16s max)
- Maintain fallback values for non-critical operations
- Log all errors with context for debugging
- Alert administrators for critical errors

## Testing Strategy

### Unit Testing Approach

Unit tests validate specific examples, edge cases, and error conditions for individual components.

**Test Coverage Areas**:
- Authentication: Valid/invalid credentials, password hashing, token generation
- Profile Management: CRUD operations, validation, data persistence
- Analytics Calculations: Engagement rate, consistency scores, growth rates
- NLP Analysis: Sentiment analysis, brand safety scoring, edge cases (empty text, special characters)
- ML Model: Feature engineering, prediction bounds, confidence scoring
- Pricing Adjustments: Duration multipliers, deliverable pricing, campaign type adjustments
- Data Validation: Input validation, range checking, format validation
- Error Handling: Error messages, recovery mechanisms, edge cases

**Unit Test Examples**:
```
test_engagement_rate_calculation_valid_input()
test_engagement_rate_calculation_zero_followers()
test_engagement_rate_calculation_zero_interactions()
test_profile_update_preserves_unmodified_fields()
test_password_hashing_produces_different_hashes()
test_sentiment_analysis_positive_text()
test_sentiment_analysis_negative_text()
test_sentiment_analysis_empty_text()
test_ml_model_output_bounds()
test_pricing_adjustment_for_duration()
test_pricing_adjustment_for_deliverables()
test_invalid_email_format_rejected()
test_negative_follower_count_rejected()
```

### Property-Based Testing Approach

Property-based testing validates universal properties across many generated inputs using randomization.

**Property Test Configuration**:
- Minimum 100 iterations per property test
- Each test references its design document property
- Tag format: `Feature: brand-deal-price-estimator, Property {number}: {property_text}`

**Property Test Examples**:

```python
# Property 1: Authentication Round Trip
@property_test(iterations=100)
def test_auth_round_trip(email, password):
    """For any valid credentials, registration + verification + login should succeed"""
    # Generate random valid email and password
    # Register user
    # Verify email
    # Login with same credentials
    # Assert: session token is valid and non-empty
    pass

# Property 3: Engagement Rate Calculation
@property_test(iterations=100)
def test_engagement_rate_calculation(followers, interactions):
    """For any engagement metrics, calculated rate should equal formula"""
    # Generate random followers (1-10M) and interactions (0-followers)
    # Calculate engagement rate
    # Assert: rate == (interactions / followers) * 100
    # Assert: 0 <= rate <= 100
    pass

# Property 8: ML Model Output Validity
@property_test(iterations=100)
def test_ml_model_output_validity(features):
    """For any valid features, model output should have valid price range"""
    # Generate random valid features
    # Call ML model prediction
    # Assert: min_price <= recommended_price <= max_price
    # Assert: all prices > 0
    # Assert: confidence_score in [0, 100]
    pass

# Property 14: Data Encryption at Rest
@property_test(iterations=50)
def test_data_encryption_at_rest(sensitive_data):
    """For any sensitive data, stored data should be encrypted"""
    # Generate random sensitive data (passwords, tokens)
    # Store in database
    # Query raw database
    # Assert: stored data is not readable plaintext
    # Assert: decryption with correct key produces original data
    pass

# Property 17: Export Data Consistency
@property_test(iterations=100)
def test_export_data_consistency(user_id, export_format):
    """For any export, exported data should match dashboard data"""
    # Generate random user with metrics
    # Get dashboard data
    # Export to format (PDF or CSV)
    # Parse exported data
    # Assert: exported data matches dashboard data
    pass
```

### Integration Testing Approach

Integration tests verify end-to-end workflows and component interactions.

**Integration Test Scenarios**:
- User registration → email verification → login → profile creation → social media linking
- Social media linking → data fetch → analytics calculation → recommendation generation
- Recommendation generation → benchmark comparison → dashboard display → export
- Error scenarios: API failures, database errors, validation failures

### Test Execution Strategy

- Run unit tests on every code commit (fast feedback)
- Run property tests nightly (comprehensive coverage)
- Run integration tests before release (end-to-end validation)
- Maintain test coverage above 80% for critical paths
- Use continuous integration (CI) pipeline for automated testing

### Performance Testing

- Load test dashboard with 1000+ concurrent users
- Stress test ML model with 10,000+ prediction requests
- Test API response times under various load conditions
- Monitor database query performance with large datasets
- Verify caching effectiveness for frequently accessed data

### Security Testing

- SQL injection testing on all database queries
- XSS testing on all user input fields
- CSRF protection verification
- Authentication bypass attempts
- Authorization boundary testing
- Sensitive data exposure testing
