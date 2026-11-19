# FastAPI + Pydantic: Complete Notes

Notes covering FastAPI fundamentals, HTTP methods, path & query parameters, Pydantic, the Patient Management API (CRUD), serving an ML model (Insurance Premium Predictor), Dockerization, and deployment on AWS EC2.

## Table of Contents

1. [Introduction: Why FastAPI?](#1-introduction-why-fastapi)
2. [HTTP Methods & CRUD](#2-http-methods--crud)
3. [Project Overview: Patient Management System (PMS)](#3-project-overview-patient-management-system-pms)
4. [Path Parameters](#4-path-parameters)
5. [HTTP Status Codes & HTTPException](#5-http-status-codes--httpexception)
6. [Query Parameters](#6-query-parameters)
7. [Pydantic](#7-pydantic)
8. [POST: Request Body](#8-post-request-body)
9. [PUT and DELETE](#9-put-and-delete)
10. [Serving the ML Model](#10-serving-the-ml-model)
11. [Best Practices for APIs](#11-best-practices-for-apis)
12. [Dockerization](#12-dockerization)
13. [Deployment on AWS EC2](#13-deployment-on-aws-ec2)
14. [Quick Revision Cheat Sheet](#14-quick-revision-cheat-sheet)

---

## 1. Introduction: Why FastAPI?

### 1.1 Why FastAPI is fast to **run**

A request travels like this:

```
Client ──HTTP──▶ Web Server ──(SGI)──▶ API Code ──▶ response back to client
```

- **Web Server**: receives the raw HTTP request (Gunicorn / Uvicorn).
- **SGI (Server Gateway Interface)**: a *protocol* (standard) that defines how the web server talks to the Python API code. It has two flavours: **WSGI** and **ASGI**.
- **API Code**: your application logic (e.g. `/predict` endpoint that runs an ML model).

**Example: ML model API**

```
Client sends:                                   Server replies:
POST /predict HTTP/1.1                          HTTP/1.1 200 OK
Host: api.example.com                           Content-Type: application/json
Content-Type: application/json
Content-Length: 45                              { "prediction": 8.3 }

{ "feature1": 5.2, "feature2": 3.1 }
```

Inside the API code, the request is available as:

```
Request.method  --> "POST"
Request.url     --> "/predict"
Request.json()  --> {"feature1": 5.2, "feature2": 3.1}
```

The `/predict` endpoint takes features (f1, f2), runs the model and returns the **prediction** (this model API can be hosted on AWS).

### 1.2 WSGI vs ASGI

| | **Flask** | **FastAPI** |
|---|---|---|
| Protocol | **WSGI** (Web Server Gateway Interface) | **ASGI** (Asynchronous Server Gateway Interface) |
| Web server | **Gunicorn** | **Uvicorn** |
| Toolkit underneath | **Werkzeug** | **Starlette** |
| Endpoint type | Synchronous | Asynchronous |

**Synchronous endpoint (Flask)**

```python
@app.route("/predict", methods=["POST"])
def predict():
    json_data = request.get_json()
    data = InputData(**json_data)
    result = predict_sync(data)
    return jsonify(result)
```

**Asynchronous endpoint (FastAPI)**

```python
@app.post("/predict")
async def predict(data: InputData):
    result = await predict_async(data)
    return result
```

**Key idea:** with `async`/`await`, while one request is waiting (for I/O, a DB, a model call, etc.), the server can handle other requests instead of blocking. That is why FastAPI is **fast to run**.

### 1.3 Why FastAPI is fast to **code**

1. **Automatic input validation** (via Pydantic and type hints).
2. **Auto-generated interactive documentation** (Swagger UI at `/docs`, ReDoc at `/redoc`).
3. **Seamless integration with the modern ecosystem**: ML/DL libraries, OAuth, JWT, SQLAlchemy, Docker, Kubernetes, etc.

---

## 2. HTTP Methods & CRUD

### 2.1 Static vs Dynamic software

- **Static**: content does not change per request (e.g. a calendar view, blogs, government websites).
- **Dynamic**: content is created, read, updated and deleted by users (e.g. a spreadsheet/Excel, apps with profiles).

### 2.2 CRUD

| Letter | Operation | HTTP verb |
|---|---|---|
| **C** | Create | `POST` |
| **R** | Retrieve / Read | `GET` |
| **U** | Update | `PUT` |
| **D** | Delete | `DELETE` |

### 2.3 How the web works

```
Client ──HTTP request──▶ Server (website)
Client ◀──HTTP response── Server
```

- A **website** is a type of **software**.
- Every request has an **HTTP verb** (GET, POST, PUT, DELETE) that tells the server what the client wants to do.
- Example: a **profile** page uses `GET` to view, `POST` to create, `PUT` to update, `DELETE` to remove.

---

## 3. Project Overview: Patient Management System (PMS)

The data is stored as **JSON** (like a small database). Each patient is keyed by ID:

```json
{
  "P001": {
    "id": "P001",
    "name": "Ananya Sharma",
    "city": "Guwahati",
    "age": 28,
    "gender": "female",
    "height": 1.65,
    "weight": 90.0,
    "bmi": 33.06,
    "verdict": "Obese"
  },
  "P002": {
    "id": "P002",
    "name": "Ravi Mehta",
    "city": "Mumbai",
    "age": 35,
    "gender": "male",
    "height": 1.75,
    "weight": 85,
    "bmi": 27.76,
    "verdict": "Overweight"
  }
}
```

A front-end (e.g. a "New Patient" form with ID, Name, City, Age, Gender, Height, Weight and a SAVE button) talks to the **API**, and the API reads/writes the JSON.

### Planned endpoints

| Endpoint | Method | Purpose |
|---|---|---|
| `/` (home) | GET | Returns a hello message |
| `/about` | GET | About the API |
| `/view` | GET | View all patients |
| `/patient/{patient_id}` | GET | View one patient (**path param**) |
| `/sort` | GET | Sort patients by height / weight / bmi (**query param**) |
| `/create` | POST | Create a patient (JSON **request body**) |
| `/update/{patient_id}` | PUT | Update a patient |
| `/delete/{patient_id}` | DELETE | Delete a patient |

> **BMI** and **verdict** are *computed* from height and weight, so the user does not enter them.

---

## 4. Path Parameters

> **Path parameters are dynamic segments of a URL path used to identify a specific resource.**

```
localhost:8000/view        → all patients
localhost:8000/view/3      → specific record (patient 3)
localhost:8000/view/4      → specific record (patient 4)
```

Used to **retrieve**, **update** and **delete** a specific resource.

### Basic example

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

@app.get("/patient/{patient_id}")
def view_patient(patient_id: str):
    patients = load_data()
    if patient_id in patients:
        return patients[patient_id]
    raise HTTPException(status_code=404, detail="Patient not found")
```

### `Path()` function

`Path()` is used to provide **metadata, validation rules and documentation hints** for path parameters.

Available options: `title`, `description`, `example`, `ge`, `gt`, `le`, `lt`, `min_length`, `max_length`, `regex`.

```python
from fastapi import Path

@app.get("/patient/{patient_id}")
def view_patient(
    patient_id: str = Path(
        ...,
        title="Patient ID",
        description="ID of the patient in the DB",
        example="P001",
    )
):
    ...
```

> `...` (Ellipsis) means the parameter is **required**.

---

## 5. HTTP Status Codes & HTTPException

**HTTP status codes** are 3-digit numbers returned by a web server (like FastAPI) to indicate the **result** of a client's request (from a browser or API consumer).

They help the client (browser, frontend, mobile app, etc.) understand:

- whether the request was **successful**,
- whether something **went wrong**,
- and what **kind of issue** occurred (if any).

### Categories

| Range | Meaning | Description |
|---|---|---|
| **2xx** | Success | The request was successfully received and processed |
| **3xx** | Redirection | Further action needs to be taken (e.g. redirect) |
| **4xx** | Client Error | Something is wrong with the request from the client |
| **5xx** | Server Error | Something went wrong on the server side |

### Common codes

| Code | Name | Meaning | Example |
|---|---|---|---|
| **200** | OK | Standard success | A `GET` or `POST` succeeded |
| **201** | Created | Resource created | After a `POST` that creates something |
| **204** | No Content | Success, but no data returned | After a `DELETE` request |
| **400** | Bad Request | Malformed or invalid request | Missing field, wrong data type |
| **401** | Unauthorized | No/invalid authentication | Login required |
| **403** | Forbidden | Authenticated, but no permission | Logged in but not allowed |
| **404** | Not Found | Resource doesn't exist | Patient ID not in DB |
| **500** | Internal Server Error | Generic failure | Something broke on the server |
| **502** | Bad Gateway | Gateway (like Nginx) failed to reach backend | |
| **503** | Service Unavailable | Server is down or overloaded | |

### `HTTPException`

`HTTPException` is a special built-in exception in FastAPI used to **return custom HTTP error responses** when something goes wrong in your API.

Instead of returning a normal JSON or crashing the server, you can **gracefully raise an error** with:

- a proper **HTTP status code** (like 404, 400, 403, etc.),
- a **custom error message**,
- (optional) **extra headers**.

```python
from fastapi import HTTPException

raise HTTPException(status_code=404, detail="Patient not found")
```

---

## 6. Query Parameters

> **Query parameters are optional key-value pairs appended to the end of a URL, used to pass additional data to the server in an HTTP request. They are typically employed for operations like filtering, sorting, searching and pagination, without altering the endpoint path itself.**

```
/patients?city=Delhi&sort_by=age
```

- `?` marks the **start** of query parameters.
- Each parameter is a **key-value pair**: `key=value`.
- Multiple parameters are separated by `&`.

In the example above:

- `city=Delhi` → query parameter for **filtering**
- `sort_by=age` → query parameter for **sorting**

### Sorting patients (project example)

`/sort?sort_by=<field>&order=<asc|desc>`

- `sort_by` → `height`, `weight` or `bmi` (**required**)
- `order` → `asc` or `desc` (**optional**, default `asc`)

```python
from fastapi import Query

@app.get("/sort")
def sort_patients(
    sort_by: str = Query(..., description="Sort on the basis of height, weight or bmi"),
    order: str = Query("asc", description="Sort in asc or desc order"),
):
    valid_fields = ["height", "weight", "bmi"]

    if sort_by not in valid_fields:
        raise HTTPException(status_code=400, detail=f"Invalid field, select from {valid_fields}")
    if order not in ["asc", "desc"]:
        raise HTTPException(status_code=400, detail="Invalid order, select asc or desc")

    data = load_data()
    sort_order = True if order == "desc" else False
    sorted_data = sorted(data.values(), key=lambda x: x.get(sort_by, 0), reverse=sort_order)
    return sorted_data
```

### `Query()` function

`Query()` is a utility function provided by FastAPI to **declare, validate and document** query parameters in your API endpoints.

It allows you to:

- Set **default values**
- Enforce **validation rules**
- Add **metadata** like description, title, examples

| Option | Purpose |
|---|---|
| `default` | Set default value (e.g. `Query(0)`) |
| `title` | Displayed in API docs |
| `description` | Detailed explanation in Swagger |
| `example` / `examples` | Provide sample inputs |
| `min_length`, `max_length` | Validate string length |
| `ge`, `gt`, `le`, `lt` | Validate numeric bounds |
| `regex` | Pattern match for strings |

### Path param vs Query param

| | Path parameter | Query parameter |
|---|---|---|
| Position | Part of the URL path: `/patient/P001` | After `?`: `/sort?sort_by=bmi` |
| Required? | Always required | Usually optional |
| Used for | Identifying a **specific resource** | Filtering, sorting, searching, pagination |
| Validated with | `Path()` | `Query()` |

---

## 7. Pydantic

Pydantic is a Python library for **data validation** using type hints. FastAPI uses it under the hood for request bodies and responses.

### Workflow (3 steps)

1. **Define a Pydantic model** (a `class`) that represents the **ideal schema** of the data.
   - This includes the expected fields, their types, and any validation constraints (e.g. `gt=0` for positive numbers).
2. **Instantiate the model with raw input data** (usually a dictionary or JSON-like structure).
   - Pydantic automatically **validates** the data and **coerces** it into the correct Python types (if possible).
   - If the data doesn't meet the model's requirements, Pydantic raises a `ValidationError`.
3. **Pass the validated model object** to functions or use it throughout your codebase.
   - This ensures every part of your program works with **clean, type-safe and logically valid data**.

```
{ "name": "nitish", "age": 30 }  ──▶  Pydantic model  ──▶  Pydantic object (validated)
```

### Example

```python
from pydantic import BaseModel, Field, EmailStr
from typing import List, Dict, Optional, Annotated

class Patient(BaseModel):
    name: str
    age: int
    weight: float
    married: bool = False                      # default value
    allergies: Optional[List[str]] = None      # optional field
    contact_details: Dict[str, str]

patient_info = {
    "name": "nitish",
    "age": "30",            # string, will be coerced to int 30
    "weight": 75.2,
    "contact_details": {"phone": "123456"},
}

patient1 = Patient(**patient_info)   # validation + type coercion happens here
print(patient1.age)                  # 30 (int)
```

> Type coercion: `"30"` → `30`. If coercion is impossible (e.g. `"thirty"`), Pydantic raises `ValidationError`.

### Field(): constraints & metadata

```python
class Patient(BaseModel):
    name: Annotated[str, Field(max_length=50, title="Name of the patient",
                               description="Give the name in less than 50 chars",
                               examples=["Nitish", "Amit"])]
    email: EmailStr
    age: int = Field(gt=0, lt=120)
    weight: Annotated[float, Field(gt=0, strict=True)]
    married: Annotated[bool, Field(default=None, description="Is the patient married or not")]
```

### Custom validators

```python
from pydantic import field_validator, model_validator, computed_field

class Patient(BaseModel):
    name: str
    email: EmailStr
    age: int
    weight: float   # kg
    height: float   # m

    # validates a single field
    @field_validator("email")
    @classmethod
    def email_validator(cls, value):
        valid_domains = ["hdfc.com", "icici.com"]
        domain_name = value.split("@")[-1]
        if domain_name not in valid_domains:
            raise ValueError("Not a valid domain")
        return value

    @field_validator("name")
    @classmethod
    def transform_name(cls, value):
        return value.upper()

    # validates multiple fields together
    @model_validator(mode="after")
    def validate_emergency_contact(cls, model):
        ...
        return model

    # derived field
    @computed_field
    @property
    def bmi(self) -> float:
        return round(self.weight / (self.height ** 2), 2)
```

### Nested models & serialization

```python
class Address(BaseModel):
    city: str
    state: str
    pin: str

class Patient(BaseModel):
    name: str
    address: Address          # nested model

p = Patient(name="Ravi", address=Address(city="Mumbai", state="MH", pin="400001"))

p.model_dump()                          # → dict
p.model_dump_json()                     # → JSON string
p.model_dump(exclude_unset=True)        # only fields that were explicitly set
p.model_dump(include={"name"})          # only chosen fields
```

(Pydantic v2 method names. In v1 these were `.dict()` / `.json()`.)

---

## 8. POST: Request Body

> A **request body** is the portion of an HTTP request that contains data sent by the client to the server. It is typically used in HTTP methods such as **POST** or **PUT** to transmit structured data (e.g. JSON, XML, form-data) for the purpose of **creating** or **updating** resources on the server. The server parses the request body to extract the necessary information and perform the intended operation.

### Plan of action (create endpoint)

1. **Client → Server**: client sends an HTTP request (`POST`) with **JSON** in the **request body**.
2. **Validate** the data using a **Pydantic model**.
3. Add the **new record** to the **JSON file** (our database).

`GET` requests have no body; the data goes in the URL (path/query).

### Pydantic model for the Patient (project)

```python
from pydantic import BaseModel, Field, computed_field
from typing import Annotated, Literal

class Patient(BaseModel):
    id: Annotated[str, Field(..., description="ID of the patient", examples=["P001"])]
    name: Annotated[str, Field(..., description="Name of the patient")]
    city: Annotated[str, Field(..., description="City where the patient is living")]
    age: Annotated[int, Field(..., gt=0, lt=120, description="Age of the patient")]
    gender: Annotated[Literal["male", "female", "others"], Field(..., description="Gender of the patient")]
    height: Annotated[float, Field(..., gt=0, description="Height of the patient in mtrs")]
    weight: Annotated[float, Field(..., gt=0, description="Weight of the patient in kgs")]

    @computed_field
    @property
    def bmi(self) -> float:
        return round(self.weight / (self.height ** 2), 2)

    @computed_field
    @property
    def verdict(self) -> str:
        if self.bmi < 18.5:
            return "Underweight"
        elif self.bmi < 25:
            return "Normal"
        elif self.bmi < 30:
            return "Overweight"
        else:
            return "Obese"
```

### Helper functions for the JSON "database"

```python
import json

def load_data():
    with open("patients.json", "r") as f:
        return json.load(f)

def save_data(data):
    with open("patients.json", "w") as f:
        json.dump(data, f)
```

### `/create` endpoint

```python
from fastapi.responses import JSONResponse

@app.post("/create")
def create_patient(patient: Patient):
    data = load_data()

    # check if the patient already exists
    if patient.id in data:
        raise HTTPException(status_code=400, detail="Patient already exists")

    # add new patient to the database (pydantic object → dict)
    data[patient.id] = patient.model_dump(exclude=["id"])

    save_data(data)

    return JSONResponse(status_code=201, content={"message": "patient created successfully"})
```

Returns **201 Created**.

---

## 9. PUT and DELETE

Project progress: `view`, `patient`, `sort`, `create` are done. Remaining: **update** and **delete**, which completes CRUD.

### Update endpoint (`PUT /update/{patient_id}`)

For updates, fields should be **optional** (the client may send only the fields to change), so we define a separate Pydantic model.

```python
from typing import Optional

class PatientUpdate(BaseModel):
    name: Annotated[Optional[str], Field(default=None)]
    city: Annotated[Optional[str], Field(default=None)]
    age: Annotated[Optional[int], Field(default=None, gt=0)]
    gender: Annotated[Optional[Literal["male", "female"]], Field(default=None)]
    height: Annotated[Optional[float], Field(default=None, gt=0)]
    weight: Annotated[Optional[float], Field(default=None, gt=0)]

@app.put("/update/{patient_id}")
def update_patient(patient_id: str, patient_update: PatientUpdate):
    data = load_data()

    if patient_id not in data:
        raise HTTPException(status_code=404, detail="Patient not found")

    existing_patient_info = data[patient_id]

    # only the fields the client actually sent
    updated_patient_info = patient_update.model_dump(exclude_unset=True)

    for key, value in updated_patient_info.items():
        existing_patient_info[key] = value

    # bmi and verdict must be recalculated, so rebuild a full Patient object
    existing_patient_info["id"] = patient_id
    patient_pydantic_obj = Patient(**existing_patient_info)
    existing_patient_info = patient_pydantic_obj.model_dump(exclude=["id"])

    data[patient_id] = existing_patient_info
    save_data(data)

    return JSONResponse(status_code=200, content={"message": "patient updated"})
```

**Flow:** get existing record → merge new fields → rebuild `Patient` (to recompute `bmi`/`verdict`) → dump back to dict → save.

### Delete endpoint (`DELETE /delete/{patient_id}`)

```python
@app.delete("/delete/{patient_id}")
def delete_patient(patient_id: str):
    data = load_data()

    if patient_id not in data:
        raise HTTPException(status_code=404, detail="Patient not found")

    del data[patient_id]
    save_data(data)

    return JSONResponse(status_code=200, content={"message": "patient deleted"})
```

### All endpoints of the PMS (CRUD complete)

| Operation | Method | Endpoint |
|---|---|---|
| Create | POST | `/create` |
| Read all | GET | `/view` |
| Read one | GET | `/patient/{patient_id}` |
| Sort | GET | `/sort?sort_by=&order=` |
| Update | PUT | `/update/{patient_id}` |
| Delete | DELETE | `/delete/{patient_id}` |

---

## 10. Serving the ML Model

### Plan of attack

The ML model is **built**, **wrapped in an API**, and then used from a **frontend**.

```
 Part 1                    Part 2                Part 3
 Model Building    ──▶     FastAPI        ──▶    Streamlit
   (model.pkl)              (API)                 (Frontend)
                                                  (instead of writing HTML/CSS/JS)
```

- `model.pkl` ⇄ **API** ⇄ **Frontend**
- **Streamlit** is used to build the frontend in Python, without needing HTML/CSS/JavaScript.

### Step 1: Building & exporting the model

**Use case:** predict the **insurance premium category** (Low / Medium / High). Insurance **companies** use such models to decide the premium for a customer.

**Raw dataset columns**

| age | weight | height | income_lpa | smoker | city | occupation | insurance_premium_category |
|---|---|---|---|---|---|---|---|
| 64 | 59.8 | 1.63 | 3.87 | False | Mumbai | retired | Medium |
| 51 | 100.6 | 1.68 | 11.99 | True | Bangalore | unemployed | High |
| 40 | 70.0 | 1.59 | 28.17 | True | Bangalore | government_job | Low |

**Feature engineering (transformed data)**

| New feature | Derived from |
|---|---|
| `bmi` | weight / height² |
| `age_group` | age → young / adult / middle_aged / senior |
| `lifestyle_risk` | smoker + bmi → low / medium / high |
| `city_tier` | city → tier 1 / 2 / 3 |
| `income_lpa` | kept as is |
| `occupation` | kept as is |

Train a model (a **Random Forest**, `rf`, in this project), then **export** it:

```python
import pickle

with open("model.pkl", "wb") as f:
    pickle.dump(model, f)
```

### Step 2: Building the API endpoint

`/predict` (**POST**) takes the **request body** (JSON) with raw inputs.

```
Raw input (age, weight, height, income_lpa, smoker, city, occupation)
        │  validate with Pydantic
        ▼
Transformed data (bmi, age_group, lifestyle_risk, city_tier, income_lpa, occupation)
        │  model.predict()
        ▼
JSON response → predicted category
```

```python
import pickle
import pandas as pd
from fastapi import FastAPI
from fastapi.responses import JSONResponse
from pydantic import BaseModel, Field, computed_field
from typing import Annotated, Literal

with open("model.pkl", "rb") as f:
    model = pickle.load(f)

app = FastAPI()

tier_1_cities = ["Mumbai", "Delhi", "Bangalore", "Chennai", "Kolkata", "Hyderabad", "Pune"]
tier_2_cities = ["Jaipur", "Chandigarh", "Indore", "Lucknow", "Patna", "Ranchi", "Visakhapatnam"]

class UserInput(BaseModel):
    age: Annotated[int, Field(..., gt=0, lt=120, description="Age of the user")]
    weight: Annotated[float, Field(..., gt=0, description="Weight of the user (kg)")]
    height: Annotated[float, Field(..., gt=0, lt=2.5, description="Height of the user (m)")]
    income_lpa: Annotated[float, Field(..., gt=0, description="Annual salary in LPA")]
    smoker: Annotated[bool, Field(..., description="Is the user a smoker")]
    city: Annotated[str, Field(..., description="City the user belongs to")]
    occupation: Annotated[Literal["retired", "freelancer", "student", "government_job",
                                  "business_owner", "unemployed", "private_job"],
                          Field(..., description="Occupation of the user")]

    @computed_field
    @property
    def bmi(self) -> float:
        return self.weight / (self.height ** 2)

    @computed_field
    @property
    def lifestyle_risk(self) -> str:
        if self.smoker and self.bmi > 30:
            return "high"
        elif self.smoker or self.bmi > 27:
            return "medium"
        else:
            return "low"

    @computed_field
    @property
    def age_group(self) -> str:
        if self.age < 25:
            return "young"
        elif self.age < 45:
            return "adult"
        elif self.age < 60:
            return "middle_aged"
        return "senior"

    @computed_field
    @property
    def city_tier(self) -> int:
        if self.city in tier_1_cities:
            return 1
        elif self.city in tier_2_cities:
            return 2
        return 3

@app.post("/predict")
def predict_premium(data: UserInput):
    input_df = pd.DataFrame([{
        "bmi": data.bmi,
        "age_group": data.age_group,
        "lifestyle_risk": data.lifestyle_risk,
        "city_tier": data.city_tier,
        "income_lpa": data.income_lpa,
        "occupation": data.occupation,
    }])
    prediction = model.predict(input_df)[0]
    return JSONResponse(status_code=200, content={"predicted_category": prediction})
```

Run it with:

```bash
uvicorn app:app --reload
```

Then open `http://localhost:8000/docs` to test it in Swagger UI.

### Step 3: Frontend (Streamlit)

```python
import requests
import streamlit as st

API_URL = "http://localhost:8000/predict"

st.title("Insurance Premium Category Predictor")

age = st.number_input("Age", min_value=1, max_value=119, value=30)
weight = st.number_input("Weight (kg)", min_value=1.0, value=65.0)
height = st.number_input("Height (m)", min_value=0.5, max_value=2.5, value=1.7)
income_lpa = st.number_input("Annual Income (LPA)", min_value=0.1, value=10.0)
smoker = st.selectbox("Are you a smoker?", [True, False])
city = st.text_input("City", value="Mumbai")
occupation = st.selectbox("Occupation", ["retired", "freelancer", "student", "government_job",
                                         "business_owner", "unemployed", "private_job"])

if st.button("Predict Premium Category"):
    payload = {"age": age, "weight": weight, "height": height, "income_lpa": income_lpa,
               "smoker": smoker, "city": city, "occupation": occupation}
    try:
        response = requests.post(API_URL, json=payload)
        if response.status_code == 200:
            st.success(f"Predicted Category: **{response.json()['predicted_category']}**")
        else:
            st.error(f"API Error: {response.status_code}")
            st.write(response.text)
    except requests.exceptions.ConnectionError:
        st.error("Could not connect to the FastAPI server.")
```

Run with `streamlit run frontend.py`.

> When you deploy the API on EC2, remember to **change `API_URL`** in the frontend to the EC2 public IP.

---

## 11. Best Practices for APIs

The first version of the API is **simple, with flaws**. We improve it into an **improved API** using this checklist:

1. **Create a new folder** (proper project structure).
2. **Field validator for the `city` feature** (normalize, e.g. `"mumbai "` → `"Mumbai"`).
3. **Add routes**
   - a. **Home** (`/`)
   - b. **Health check** (`/health`)
4. **Add model version**
5. **Separation of logic**
   - a. **Pydantic model**
   - b. **City tier**
   - c. **ML logic**
6. **Try / catch** (error handling)
7. **Add confidence score**
8. **Response model**

### Suggested project structure

```
insurance-api/
├── app.py                   # FastAPI app, routes
├── schema/
│   ├── user_input.py        # Pydantic model for input
│   └── prediction_response.py   # Response model
├── config/
│   └── city_tier.py         # city tier lists
├── model/
│   ├── model.pkl
│   └── predict.py           # ML logic, model version
├── requirements.txt
└── Dockerfile
```

### 11.1 City field validator

```python
from pydantic import field_validator

@field_validator("city")
@classmethod
def normalize_city(cls, v: str) -> str:
    v = v.strip().title()
    return v
```

### 11.2 Home and health-check routes

```python
MODEL_VERSION = "1.0.0"

@app.get("/")
def home():
    return {"message": "Insurance Premium Prediction API"}

@app.get("/health")
def health_check():
    return {
        "status": "OK",
        "version": MODEL_VERSION,
        "model_loaded": model is not None,
    }
```

The health check is used by load balancers, Docker, Kubernetes and monitoring tools to know whether the app is alive.

### 11.3 Separation of logic

Keep three things in separate modules:

- **Pydantic model** → input validation and feature engineering.
- **City tier** → config / lookup lists.
- **ML logic** → loading the model, `predict()`, confidence score.

This makes the code easier to read, test and maintain.

### 11.4 Try / catch

```python
@app.post("/predict", response_model=PredictionResponse)
def predict_premium(data: UserInput):
    try:
        result = predict_output(user_input)
        return JSONResponse(status_code=200, content={"response": result})
    except Exception as e:
        return JSONResponse(status_code=500, content=str(e))
```

### 11.5 Confidence score

The model (`rf` = Random Forest) outputs class probabilities. Use them to return a **confidence** along with the prediction.

Example: for a prediction `high`, probabilities may look like

| Class | Probability |
|---|---|
| **high** | **0.73** (confidence) |
| medium | 0.17 |
| low | 0.10 |

```python
import numpy as np

class_labels = model.classes_.tolist()

def predict_output(user_input: dict):
    df = pd.DataFrame([user_input])
    predicted_class = model.predict(df)[0]

    probabilities = model.predict_proba(df)[0]
    confidence = max(probabilities)
    class_probs = dict(zip(class_labels, map(lambda p: round(p, 4), probabilities)))

    return {
        "predicted_category": predicted_class,
        "confidence": round(confidence, 4),
        "class_probabilities": class_probs,
    }
```

### 11.6 Response model

In FastAPI, a **response model** defines the **structure of the data** that your API endpoint will return. It helps in:

1. **Generating clean API docs** (`/docs`)
2. **Validating output** (so your API doesn't return malformed responses)
3. **Filtering unnecessary data** from the response

```python
from pydantic import BaseModel, Field
from typing import Dict

class PredictionResponse(BaseModel):
    predicted_category: str = Field(..., description="The predicted insurance premium category",
                                    examples=["High"])
    confidence: float = Field(..., description="Model's confidence for predicted class (0 to 1)",
                              examples=[0.8432])
    class_probabilities: Dict[str, float] = Field(
        ..., description="Probability distribution across all possible classes",
        examples=[{"High": 0.85, "Medium": 0.12, "Low": 0.03}],
    )
```

> **Two directions of Pydantic in an API:**
> - **Input** → validate user data (request body)
> - **Output** → validate API response (response model)

---

## 12. Dockerization

### Why Docker?

Your project is: ML model → prediction API (FastAPI `/predict` endpoint). To run it on any machine without "it works on my laptop" problems, we **Dockerize** it: package code + dependencies + runtime into a **Docker image**.

### Setup

1. Install **Docker** (Docker Desktop).
2. Create an account on **Docker Hub**.

### Steps to create a Docker image

**Step 1: Create a `Dockerfile`**

```dockerfile
# Base image
FROM python:3.11-slim

# Set the working directory inside the container
WORKDIR /app

# Copy and install dependencies first (better caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of the application
COPY . .

# Expose the port
EXPOSE 8000

# Run the app
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

> `--host 0.0.0.0` is required so the app is reachable from outside the container.

**Step 2: Build the Docker image**

```bash
docker build -t tweakster24/insurancepremium-api .
```

**Step 3: Login to Docker Hub**

```bash
docker login
```

**Step 4: Push the image to Docker Hub**

```bash
docker push tweakster24/insurancepremium-api
```

**Step 5: Pull the Docker image** (on any machine)

```bash
docker pull tweakster24/insurance-premium-api
```

**Step 6: Run the Docker image locally**

```bash
docker run -p 8000:8000 tweakster24/insurance-premium-api
```

> `-p 8000:8000` maps **host port : container port**.
>
> ⚠️ **Note:** the image name must be **exactly the same** in build, push, pull and run commands. In the original notes the name appears as both `insurancepremium-api` and `insurance-premium-api`. Pick one spelling and use it everywhere.

### Useful Docker commands

```bash
docker images                 # list images
docker ps                     # running containers
docker ps -a                  # all containers
docker stop <container_id>    # stop a container
docker rm <container_id>      # remove a container
docker rmi <image_id>         # remove an image
```

---

## 13. Deployment on AWS EC2

### Steps for deployment

1. **Create an EC2 instance** (Ubuntu).
2. **Connect** to the EC2 instance (SSH / EC2 Instance Connect).
3. Run the following commands to install Docker:
   ```bash
   sudo apt-get update
   sudo apt-get install -y docker.io
   sudo systemctl start docker
   sudo systemctl enable docker
   sudo usermod -aG docker $USER
   exit
   ```
4. **Restart a new connection** to the EC2 instance (so the group change takes effect and you can run `docker` without `sudo`).
5. Pull and run the image:
   ```bash
   docker pull tweakster24/insurance-premium-api:latest
   docker run -p 8000:8000 tweakster24/insurance-premium-api
   ```
6. **Change security group settings**: add an **inbound rule** to allow custom TCP on port **8000** (source `0.0.0.0/0` for testing).
7. **Check the API**: open `http://<EC2-public-IP>:8000/docs`.
8. **Change the frontend code**: update `API_URL` to `http://<EC2-public-IP>:8000/predict`.

### Full deployment flow

```
Local code ─▶ Dockerfile ─▶ docker build ─▶ docker push ─▶ Docker Hub
                                                               │
                                                      docker pull (on EC2)
                                                               │
                                                      docker run -p 8000:8000
                                                               │
                                       Streamlit frontend ─▶ http://<EC2-IP>:8000/predict
```

---

## 14. Quick Revision Cheat Sheet

| Concept | One-line summary |
|---|---|
| FastAPI | Modern, fast, async Python web framework built on Starlette + Pydantic |
| WSGI / ASGI | Protocols between web server and app; WSGI = sync (Flask/Gunicorn), ASGI = async (FastAPI/Uvicorn) |
| `async` / `await` | Lets the server handle other requests while waiting on I/O |
| CRUD | Create (POST), Read (GET), Update (PUT), Delete (DELETE) |
| Path param | `/patient/{id}`, identifies a **specific resource**, validated with `Path()` |
| Query param | `?key=value&...`, optional filtering/sorting/pagination, validated with `Query()` |
| Request body | JSON sent in POST/PUT, parsed into a Pydantic model |
| Pydantic | Data validation + type coercion via models; raises `ValidationError` |
| `Field()` | Add constraints/metadata to model fields |
| `field_validator` | Custom validation/transformation for one field |
| `model_validator` | Validation across multiple fields |
| `computed_field` | Derived fields (e.g. `bmi`, `verdict`) |
| `HTTPException` | Raise custom HTTP errors (404, 400, ...) |
| Status codes | 2xx success, 3xx redirect, 4xx client error, 5xx server error |
| 200 / 201 / 204 | OK / Created (POST) / No Content (DELETE) |
| 400 / 401 / 403 / 404 | Bad request / Unauthorized / Forbidden / Not found |
| 500 / 502 / 503 | Internal error / Bad gateway / Service unavailable |
| Response model | Defines & validates the output structure, cleans docs, filters data |
| Confidence score | Max of `predict_proba` for the predicted class |
| Docker | Package app + dependencies into an image that runs anywhere |
| EC2 | AWS virtual machine used to host the Docker container |
| Security group | AWS firewall; open port 8000 to reach the API |

### Typical learning order

1. FastAPI intro → 2. HTTP methods (GET) → 3. Path & query params → 4. Pydantic → 5. POST → 6. PUT & DELETE → 7. Serving the ML model → 8. Best practices → 9. Dockerization → 10. Deployment

---

*Notes compiled from the FastAPI course handwritten/typed notes. Code snippets expand on the concepts in the notes and use Pydantic v2 syntax.*
