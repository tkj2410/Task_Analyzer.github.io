# Smart Task Analyzer

A Django-based web application that intelligently prioritizes tasks using a custom scoring algorithm. This system helps users identify which tasks to work on first by analyzing multiple factors including urgency, importance, effort, and dependencies.

## 🎯 Overview

The Smart Task Analyzer takes a list of tasks and calculates priority scores based on:
- **Urgency**: How soon the task is due
- **Importance**: User-provided rating (1-10 scale)
- **Effort**: Estimated hours to complete
- **Dependencies**: Tasks that block other tasks

## 🚀 Setup Instructions

### Prerequisites
- Python 3.8 or higher
- pip (Python package manager)
- Git

### Installation Steps

1. **Clone the repository**
```bash
git clone <your-repository-url>
cd task-analyzer
```

2. **Create and activate virtual environment**
```bash
# Windows
python -m venv venv
venv\Scripts\activate

# Mac/Linux
python3 -m venv venv
source venv/bin/activate
```

3. **Install dependencies**
```bash
pip install -r requirements.txt
```

4. **Navigate to backend directory**
```bash
cd backend
```

5. **Run migrations**
```bash
python manage.py makemigrations
python manage.py migrate
```

6. **Run tests**
```bash
python manage.py test tasks
```

7. **Start the development server**
```bash
python manage.py runserver
```

8. **Open the frontend**
- Open `frontend/index.html` in your web browser
- Or use a local server: `python -m http.server 8080` from the frontend directory

The API will be available at `http://127.0.0.1:8000/api/tasks/`

## 📋 API Endpoints

### 1. Analyze Tasks
**POST** `/api/tasks/analyze/`

Accepts a list of tasks and returns them sorted by priority score.

**Request Body:**
```json
{
  "tasks": [
    {
      "title": "Fix login bug",
      "due_date": "2025-12-05",
      "estimated_hours": 3,
      "importance": 8,
      "dependencies": []
    }
  ],
  "strategy": "smart"
}
```

**Response:**
```json
{
  "tasks": [
    {
      "title": "Fix login bug",
      "due_date": "2025-12-05",
      "estimated_hours": 3,
      "importance": 8,
      "dependencies": [],
      "priority_score": 85.5,
      "priority_level": "HIGH",
      "explanation": "Due in 3 days (+40 urgency) | High importance (8/10)"
    }
  ],
  "circular_dependencies": [],
  "strategy_used": "smart"
}
```

### 2. Suggest Tasks
**POST** `/api/tasks/suggest/`

Returns the top 3 tasks to work on today with explanations.

**Request Body:** Same as analyze endpoint

**Response:**
```json
{
  "suggestions": [
    {
      "rank": 1,
      "task": { /* task object with score */ },
      "reason": "Score: 125.0 - OVERDUE by 2 days (+110 urgency) | High importance (9/10)"
    }
  ]
}
```

## 🧮 Algorithm Explanation

### Core Scoring Logic

The priority scoring algorithm uses a multi-factor approach to calculate task priority. The base formula varies by strategy, but the **Smart Balance** strategy (default) works as follows:

**Formula:**
```
priority_score = urgency_score + (importance × 5) + dependency_bonus - (effort × 2)
```

### Component Breakdown

#### 1. Urgency Score (0-125 points)
The urgency component heavily weights how soon a task is due:

- **Overdue tasks**: 100 + (days_overdue × 5)
  - Example: 5 days overdue = 125 points
  - Rationale: Overdue tasks receive maximum priority with increasing weight
  
- **Due today**: 80 points
  - Critical urgency for same-day deadlines
  
- **Due tomorrow**: 60 points
  - High urgency for next-day deadlines
  
- **Due in 2-3 days**: 40 points
  - Moderate urgency for upcoming deadlines
  
- **Due this week (4-7 days)**: 20 points
  - Lower but still notable urgency
  
- **Due later**: max(0, 100 - days_until_due × 2)
  - Diminishing urgency for distant deadlines

**Design Decision**: Overdue tasks get exponentially higher scores to ensure they're addressed first. The multiplier ensures that severely overdue tasks (10+ days) receive attention even if they have lower importance.

#### 2. Importance Weight (5-50 points)
User-defined importance (1-10 scale) is multiplied by 5:

```
importance_weight = importance × 5
```

- Importance 10: +50 points
- Importance 5: +25 points
- Importance 1: +5 points

**Design Decision**: The 5× multiplier balances importance against urgency. A maximally important task (10) can overcome moderate urgency differences but won't override severely overdue tasks.

#### 3. Effort Penalty (-2 to -20 points)
Lower effort tasks receive a "quick win" bonus:

```
effort_penalty = estimated_hours × 2
```

- 1 hour task: -2 points
- 5 hour task: -10 points
- 10 hour task: -20 points

**Design Decision**: The penalty is relatively small (2× multiplier) because we don't want to always prioritize easy tasks over important ones. However, when two tasks have similar urgency and importance, the shorter task wins—encouraging momentum through quick completions.

#### 4. Dependency Bonus (0-50+ points)
Tasks that block other tasks receive priority:

```
dependency_bonus = number_of_blocked_tasks × 10
```

- Blocks 1 task: +10 points
- Blocks 5 tasks: +50 points

**Design Decision**: Each blocked task adds 10 points because completing blocking tasks unlocks parallel work. This is crucial for team productivity.

### Alternative Strategies

#### Fastest Wins
```
score = urgency_score + (importance × 2) - (effort × 10)
```
Maximizes the effort penalty to prioritize quick completions. Useful for building momentum or when context-switching is expensive.

#### High Impact
```
score = (importance × 10) + urgency_score
```
Doubles the importance weight, making it the dominant factor. Best for strategic work where importance outweighs deadlines.

#### Deadline Driven
```
score = urgency_score + (importance × 2)
```
Minimizes other factors to focus almost entirely on due dates. Useful during crunch periods or for deadline-critical work.

### Edge Case Handling

#### Missing Data
- Missing `importance`: defaults to 5 (medium)
- Missing `estimated_hours`: defaults to 1
- Missing `dependencies`: defaults to empty array []
- Missing `due_date`: defaults to today

**Rationale**: Graceful degradation ensures the system remains functional even with incomplete data.

#### Circular Dependencies
The algorithm detects circular dependencies using depth-first search (DFS):

```python
def detect_circular_dependencies(tasks):
    # Builds adjacency list and checks for cycles
    # Returns list of task indices involved in cycles
```

Tasks involved in circular dependencies are flagged but still scored normally, allowing users to address the dependency issue.

#### Invalid Dates
- Past dates are handled as overdue (not errors)
- Weekend/holiday awareness is a future enhancement

## 🎨 Design Decisions

### Why Separate `scoring.py`?
The scoring logic is isolated in its own module for:
- **Testability**: Easy to write unit tests
- **Reusability**: Can be imported anywhere
- **Maintainability**: Business logic separated from HTTP concerns

### Why Multiple Strategies?
Different contexts require different prioritization:
- **Sprint planning**: Use "High Impact"
- **End of week cleanup**: Use "Fastest Wins"
- **Crisis mode**: Use "Deadline Driven"
- **Normal work**: Use "Smart Balance"

### Frontend Technology Choice
Vanilla JavaScript was chosen over frameworks because:
- No build process needed
- Faster iteration for a small project
- Demonstrates fundamental skills
- Easy for reviewers to understand

## ⏱️ Time Breakdown

| Phase | Estimated | Actual |
|-------|-----------|--------|
| Project Setup | 5 min | 10 min |
| Backend Models & Setup | 30 min | 35 min |
| Scoring Algorithm | 30 min | 40 min |
| API Views | 30 min | 25 min |
| Unit Tests | 30 min | 25 min |
| Frontend HTML/CSS | 30 min | 25 min |
| Frontend JavaScript | 30 min | 35 min |
| Testing & Debugging | 30 min | 40 min |
| Documentation | 30 min | 15 min |
| **Total** | **4h 05min** | **4h 10min** |

## 🧪 Running Tests

```bash
# Run all tests
python manage.py test tasks

# Run specific test
python manage.py test tasks.tests.ScoringAlgorithmTests.test_overdue_task_high_priority

# Run with verbosity
python manage.py test tasks -v 2
```

### Test Coverage
- ✅ Overdue task prioritization
- ✅ Missing data handling
- ✅ Circular dependency detection
- ✅ Multiple strategies
- ✅ Edge cases (zero effort, invalid dates)

## 🎁 Bonus Features Attempted

- ✅ **Multiple Sorting Strategies**: 4 different prioritization modes
- ✅ **Circular Dependency Detection**: DFS-based cycle detection
- ✅ **Comprehensive Unit Tests**: 5+ test cases covering edge cases
- ⬜ **Eisenhower Matrix View**: Not implemented (would take ~45 min)
- ⬜ **Date Intelligence**: Weekend/holiday awareness (would take ~30 min)

## 🔮 Future Improvements

Given more time, I would implement:

1. **User Preferences System**
   - Allow users to customize scoring weights
   - Save preferred strategies per user
   - Time: ~2 hours

2. **Task History & Learning**
   - Track which suggested tasks were actually completed
   - Adjust algorithm based on user behavior
   - ML-based personalization
   - Time: ~4 hours

3. **Visual Dependency Graph**
   - Interactive graph showing task relationships
   - Highlight critical path
   - Use D3.js or similar
   - Time: ~3 hours

4. **Calendar Integration**
   - Sync with Google Calendar
   - Consider existing commitments
   - Smart scheduling suggestions
   - Time: ~3 hours

5. **Team Collaboration Features**
   - Shared task lists
   - Assignment and delegation
   - Real-time updates (WebSockets)
   - Time: ~6 hours

6. **Mobile App**
   - React Native or Flutter
   - Push notifications for deadlines
   - Time: ~40 hours

7. **Advanced Analytics**
   - Completion rate tracking
   - Productivity insights
   - Time estimation accuracy
   - Time: ~4 hours

## 📁 Project Structure

```
task-analyzer/
├── backend/
│   ├── backend/
│   │   ├── __init__.py
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   ├── tasks/
│   │   ├── __init__.py
│   │   ├── admin.py
│   │   ├── apps.py
│   │   ├── models.py          # Task data model
│   │   ├── scoring.py         # Priority algorithm
│   │   ├── views.py           # API endpoints
│   │   ├── urls.py            # URL routing
│   │   └── tests.py           # Unit tests
│   ├── manage.py
│   └── db.sqlite3
├── frontend/
│   ├── index.html             # Main interface
│   ├── styles.css             # Styling
│   └── script.js              # API integration
├── requirements.txt
└── README.md
```

## 🛠️ Technologies Used

- **Backend**: Python 3.13, Django 5.1.3
- **API**: Django REST Framework 3.15.2
- **Frontend**: HTML5, CSS3, Vanilla JavaScript
- **Database**: SQLite3
- **CORS**: django-cors-headers 4.6.0

## 📝 Sample Usage

1. **Add tasks manually** using the form, or
2. **Paste JSON array** of tasks:

```json
[
  {
    "title": "Fix critical bug",
    "due_date": "2025-12-02",
    "estimated_hours": 3,
    "importance": 9,
    "dependencies": []
  },
  {
    "title": "Write tests",
    "due_date": "2025-12-05",
    "estimated_hours": 4,
    "importance": 7,
    "dependencies": [0]
  }
]
```

3. **Select strategy** from dropdown
4. **Click "Analyze Tasks"**
5. **View prioritized results** with explanations

## 🤝 Contributing

This is an internship assignment project. However, if you'd like to suggest improvements:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## 📄 License

This project is created as part of an internship application for Singularium.

## 👤 Author

**Tarun Jena**
- GitHub: [@tkj2410](https://github.com/tkj2410)
- Email: tkj2410@gmail.com
- LinkedIn: [@tkj2410](https://linkedin.com/in/tkj2410)

## 🙏 Acknowledgments

- Django documentation for excellent API references
- Stack Overflow community for troubleshooting help

---
