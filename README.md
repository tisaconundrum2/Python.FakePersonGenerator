# Random Person Generator API

Let's build a RESTful API that generates random people with addresses and cars using FastAPI, SQLAlchemy, and Pydantic following the MVC pattern with a Repository interface.

## Project Structure

```
app/
├── main.py
├── models/
│   ├── __init__.py
│   └── models.py
├── schemas/
│   ├── __init__.py
│   └── schemas.py
├── repositories/
│   ├── __init__.py
│   └── repository.py
├── controllers/
│   ├── __init__.py
│   └── controllers.py
├── services/
│   ├── __init__.py
│   └── services.py
└── database.py
```

## Database Configuration (database.py)

```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

SQLALCHEMY_DATABASE_URL = "sqlite:///./person_generator.db"

engine = create_engine(
    SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False}
)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

## Models (models/models.py)

```python
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship

from app.database import Base

class Person(Base):
    __tablename__ = "people"
    
    id = Column(Integer, primary_key=True, index=True)
    first_name = Column(String, nullable=False)
    last_name = Column(String, nullable=False)
    ssn = Column(String, unique=True, nullable=False)
    address_id = Column(Integer, ForeignKey("addresses.id"))
    car_id = Column(Integer, ForeignKey("cars.id"))
    
    address = relationship("Address", back_populates="people")
    car = relationship("Car", back_populates="people")

class Address(Base):
    __tablename__ = "addresses"
    
    id = Column(Integer, primary_key=True, index=True)
    street = Column(String, nullable=False)
    city = Column(String, nullable=False)
    state = Column(String, nullable=False)
    postal_code = Column(String, nullable=False)
    country = Column(String, default="USA")
    
    people = relationship("Person", back_populates="address")

class Car(Base):
    __tablename__ = "cars"
    
    id = Column(Integer, primary_key=True, index=True)
    make = Column(String, nullable=False)
    model = Column(String, nullable=False)
    color = Column(String, nullable=False)
    year = Column(Integer, nullable=False)
    
    people = relationship("Person", back_populates="car")
```

## Schemas (schemas/schemas.py)

```python
from pydantic import BaseModel
from typing import Optional, List

# Car schemas
class CarBase(BaseModel):
    make: str
    model: str
    color: str
    year: int

class CarCreate(CarBase):
    pass

class Car(CarBase):
    id: int
    
    class Config:
        orm_mode = True

# Address schemas
class AddressBase(BaseModel):
    street: str
    city: str
    state: str
    postal_code: str
    country: str = "USA"

class AddressCreate(AddressBase):
    pass

class Address(AddressBase):
    id: int
    
    class Config:
        orm_mode = True

# Person schemas
class PersonBase(BaseModel):
    first_name: str
    last_name: str
    ssn: str

class PersonCreate(PersonBase):
    pass

class Person(PersonBase):
    id: int
    address_id: int
    car_id: int
    
    class Config:
        orm_mode = True

class PersonDetail(Person):
    address: Address
    car: Car
    
    class Config:
        orm_mode = True
```

## Repository Interface (repositories/repository.py)

```python
from abc import ABC, abstractmethod
from typing import List, Optional, Generic, TypeVar, Type
from pydantic import BaseModel
from sqlalchemy.orm import Session

from app.database import Base

T = TypeVar('T', bound=Base)

class Repository(Generic[T], ABC):
    @abstractmethod
    def get(self, db: Session, id: int) -> Optional[T]:
        pass
    
    @abstractmethod
    def get_all(self, db: Session, skip: int = 0, limit: int = 100) -> List[T]:
        pass
    
    @abstractmethod
    def create(self, db: Session, obj_in) -> T:
        pass
    
    @abstractmethod
    def update(self, db: Session, id: int, obj_in) -> T:
        pass
    
    @abstractmethod
    def delete(self, db: Session, id: int) -> T:
        pass

class RepositoryImpl(Repository[T]):
    def __init__(self, model: Type[T]):
        self.model = model
    
    def get(self, db: Session, id: int) -> Optional[T]:
        return db.query(self.model).filter(self.model.id == id).first()
    
    def get_all(self, db: Session, skip: int = 0, limit: int = 100) -> List[T]:
        return db.query(self.model).offset(skip).limit(limit).all()
    
    def create(self, db: Session, obj_in) -> T:
        obj_data = dict(obj_in)
        db_obj = self.model(**obj_data)
        db.add(db_obj)
        db.commit()
        db.refresh(db_obj)
        return db_obj
    
    def update(self, db: Session, id: int, obj_in) -> T:
        db_obj = db.query(self.model).filter(self.model.id == id).first()
        obj_data = dict(obj_in)
        
        for key, value in obj_data.items():
            setattr(db_obj, key, value)
        
        db.add(db_obj)
        db.commit()
        db.refresh(db_obj)
        return db_obj
    
    def delete(self, db: Session, id: int) -> T:
        db_obj = db.query(self.model).filter(self.model.id == id).first()
        db.delete(db_obj)
        db.commit()
        return db_obj

from app.models.models import Person, Address, Car

class PersonRepository(RepositoryImpl[Person]):
    def __init__(self):
        super().__init__(Person)

class AddressRepository(RepositoryImpl[Address]):
    def __init__(self):
        super().__init__(Address)

class CarRepository(RepositoryImpl[Car]):
    def __init__(self):
        super().__init__(Car)
```

## Services (services/services.py)

```python
import random
from faker import Faker
from typing import List, Optional
from sqlalchemy.orm import Session

from app.repositories.repository import PersonRepository, AddressRepository, CarRepository
from app.models.models import Person, Address, Car
from app.schemas.schemas import PersonCreate, AddressCreate, CarCreate

fake = Faker()

class PersonService:
    def __init__(self):
        self.person_repository = PersonRepository()
        self.address_repository = AddressRepository()
        self.car_repository = CarRepository()
    
    def create_random_person(self, db: Session):
        # Create a random address
        address_data = {
            "street": fake.street_address(),
            "city": fake.city(),
            "state": fake.state_abbr(),
            "postal_code": fake.zipcode(),
            "country": "USA"
        }
        address = self.address_repository.create(db, AddressCreate(**address_data))
        
        # Create a random car
        car_makes = ["Toyota", "Honda", "Ford", "Chevrolet", "BMW", "Tesla", "Audi"]
        car_models = {
            "Toyota": ["Camry", "Corolla", "RAV4", "Highlander"],
            "Honda": ["Civic", "Accord", "CR-V", "Pilot"],
            "Ford": ["F-150", "Mustang", "Explorer", "Escape"],
            "Chevrolet": ["Silverado", "Malibu", "Equinox", "Tahoe"],
            "BMW": ["3 Series", "5 Series", "X3", "X5"],
            "Tesla": ["Model 3", "Model Y", "Model S", "Model X"],
            "Audi": ["A4", "A6", "Q5", "Q7"]
        }
        car_colors = ["Red", "Blue", "Black", "White", "Silver", "Green", "Yellow"]
        
        make = random.choice(car_makes)
        model = random.choice(car_models[make])
        color = random.choice(car_colors)
        
        car_data = {
            "make": make,
            "model": model,
            "color": color,
            "year": random.randint(2010, 2025)
        }
        car = self.car_repository.create(db, CarCreate(**car_data))
        
        # Create a random person
        person_data = {
            "first_name": fake.first_name(),
            "last_name": fake.last_name(),
            "ssn": fake.ssn(),
            "address_id": address.id,
            "car_id": car.id
        }
        person = self.person_repository.create(db, PersonCreate(**person_data))
        
        return person
    
    def get_person(self, db: Session, person_id: int):
        return self.person_repository.get(db, person_id)
    
    def get_people(self, db: Session, skip: int = 0, limit: int = 100):
        return self.person_repository.get_all(db, skip, limit)
    
    def delete_person(self, db: Session, person_id: int):
        person = self.get_person(db, person_id)
        
        # Get related objects to clean them up too
        address_id = person.address_id
        car_id = person.car_id
        
        # Delete person first (due to foreign key constraints)
        deleted_person = self.person_repository.delete(db, person_id)
        
        # Then delete address and car
        self.address_repository.delete(db, address_id)
        self.car_repository.delete(db, car_id)
        
        return deleted_person
```

## Controllers (controllers/controllers.py)

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List

from app.database import get_db
from app.services.services import PersonService
from app.schemas.schemas import Person, PersonDetail

router = APIRouter()
person_service = PersonService()

@router.post("/people/random", response_model=Person, summary="Generate a random person")
def create_random_person(db: Session = Depends(get_db)):
    """
    Generate a random person with address and car
    """
    return person_service.create_random_person(db)

@router.get("/people", response_model=List[Person], summary="Get all people")
def get_people(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    """
    Get all people
    """
    people = person_service.get_people(db, skip, limit)
    return people

@router.get("/people/{person_id}", response_model=PersonDetail, summary="Get person by ID")
def get_person(person_id: int, db: Session = Depends(get_db)):
    """
    Get a specific person by ID
    """
    person = person_service.get_person(db, person_id)
    if not person:
        raise HTTPException(status_code=404, detail="Person not found")
    return person

@router.delete("/people/{person_id}", response_model=Person, summary="Delete a person")
def delete_person(person_id: int, db: Session = Depends(get_db)):
    """
    Delete a person (and associated address and car)
    """
    person = person_service.get_person(db, person_id)
    if not person:
        raise HTTPException(status_code=404, detail="Person not found")
    return person_service.delete_person(db, person_id)
```

## Main Application (main.py)

```python
import uvicorn
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.database import engine, Base
from app.controllers.controllers import router

# Create the database tables
Base.metadata.create_all(bind=engine)

app = FastAPI(title="Random Person Generator API", 
              description="An API for generating random people with addresses and cars")

# Configure CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Include routers
app.include_router(router, prefix="/api", tags=["people"])

@app.get("/")
def read_root():
    return {"message": "Welcome to Random Person Generator API"}

if __name__ == "__main__":
    uvicorn.run("app.main:app", host="0.0.0.0", port=8000, reload=True)
```

## How to Run

1. Install dependencies:
```bash
pip install fastapi uvicorn sqlalchemy faker pydantic
```

2. Run the application:
```bash
python -m app.main
```

3. Access the API documentation at http://localhost:8000/docs

## API Usage Examples

1. Generate a random person:
   - `POST /api/people/random`

2. Get all people:
   - `GET /api/people`

3. Get a specific person with their address and car:
   - `GET /api/people/{person_id}`

4. Delete a person (and their associated address and car):
   - `DELETE /api/people/{person_id}`

This implementation follows the MVC pattern with a Repository interface for data access, uses SQLAlchemy for ORM, Pydantic for data validation, and FastAPI for the REST API. The service also generates realistic random data for people, addresses, and cars using the Faker library.


---

# Adding Templates to FastAPI Random Person Generator

Let's extend this MVC application to include HTML templates similar to Flask/C# patterns. We'll add a web UI alongside the REST API.

## Updated Project Structure

```
app/
├── main.py
├── models/
│   ├── __init__.py
│   └── models.py
├── schemas/
│   ├── __init__.py
│   └── schemas.py
├── repositories/
│   ├── __init__.py
│   └── repository.py
├── controllers/
│   ├── __init__.py
│   ├── api_controllers.py
│   └── view_controllers.py
├── services/
│   ├── __init__.py
│   └── services.py
├── database.py
├── templates/
│   ├── base.html
│   ├── index.html
│   ├── people_list.html
│   ├── person_detail.html
│   └── person_form.html
└── static/
    └── style.css
```

## Updated Main Application (main.py)

```python
import uvicorn
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles
from fastapi.middleware.cors import CORSMiddleware
import os

from app.database import engine, Base
from app.controllers.api_controllers import router as api_router
from app.controllers.view_controllers import router as view_router

# Create the database tables
Base.metadata.create_all(bind=engine)

app = FastAPI(
    title="Random Person Generator API",
    description="An API for generating random people with addresses and cars"
)

# Configure CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Mount static files
static_dir = os.path.join(os.path.dirname(__file__), "static")
if os.path.exists(static_dir):
    app.mount("/static", StaticFiles(directory=static_dir), name="static")

# Include routers
app.include_router(api_router, prefix="/api", tags=["API - People"])
app.include_router(view_router, tags=["Web - Views"])

@app.get("/")
def read_root():
    return {"message": "Welcome to Random Person Generator API"}

if __name__ == "__main__":
    uvicorn.run("app.main:app", host="0.0.0.0", port=8000, reload=True)
```

## API Controllers (controllers/api_controllers.py)

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List

from app.database import get_db
from app.services.services import PersonService
from app.schemas.schemas import Person, PersonDetail

router = APIRouter()
person_service = PersonService()

@router.post("/people/random", response_model=Person, 
             summary="Generate a random person")
def create_random_person(db: Session = Depends(get_db)):
    """
    Generate a random person with address and car
    """
    return person_service.create_random_person(db)

@router.get("/people", response_model=List[Person], 
            summary="Get all people")
def get_people(skip: int = 0, limit: int = 100, 
               db: Session = Depends(get_db)):
    """
    Get all people
    """
    people = person_service.get_people(db, skip, limit)
    return people

@router.get("/people/{person_id}", response_model=PersonDetail, 
            summary="Get person by ID")
def get_person(person_id: int, db: Session = Depends(get_db)):
    """
    Get a specific person by ID
    """
    person = person_service.get_person(db, person_id)
    if not person:
        raise HTTPException(status_code=404, detail="Person not found")
    return person

@router.delete("/people/{person_id}", response_model=Person, 
               summary="Delete a person")
def delete_person(person_id: int, db: Session = Depends(get_db)):
    """
    Delete a person (and associated address and car)
    """
    person = person_service.get_person(db, person_id)
    if not person:
        raise HTTPException(status_code=404, detail="Person not found")
    return person_service.delete_person(db, person_id)
```

## View Controllers (controllers/view_controllers.py)

```python
from fastapi import APIRouter, Depends, HTTPException
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates
from fastapi import Request
from sqlalchemy.orm import Session
import os

from app.database import get_db
from app.services.services import PersonService

# Setup templates directory
templates_dir = os.path.join(os.path.dirname(__file__), "..", "templates")
templates = Jinja2Templates(directory=templates_dir)

router = APIRouter()
person_service = PersonService()

@router.get("/", response_class=HTMLResponse)
def home(request: Request):
    """
    Home page
    """
    return templates.TemplateResponse("index.html", {"request": request})

@router.get("/people", response_class=HTMLResponse)
def list_people(request: Request, skip: int = 0, limit: int = 100, 
                db: Session = Depends(get_db)):
    """
    List all people
    """
    people = person_service.get_people(db, skip, limit)
    return templates.TemplateResponse(
        "people_list.html", 
        {
            "request": request,
            "people": people,
            "skip": skip,
            "limit": limit
        }
    )

@router.get("/people/{person_id}", response_class=HTMLResponse)
def view_person(person_id: int, request: Request, 
                db: Session = Depends(get_db)):
    """
    View a specific person with details
    """
    person = person_service.get_person(db, person_id)
    if not person:
        raise HTTPException(status_code=404, detail="Person not found")
    
    return templates.TemplateResponse(
        "person_detail.html",
        {
            "request": request,
            "person": person
        }
    )

@router.get("/generate", response_class=HTMLResponse)
def generate_form(request: Request):
    """
    Show form to generate random person
    """
    return templates.TemplateResponse("person_form.html", {"request": request})

@router.post("/generate", response_class=HTMLResponse)
def generate_person(request: Request, db: Session = Depends(get_db)):
    """
    Generate a random person and redirect
    """
    person = person_service.create_random_person(db)
    
    return templates.TemplateResponse(
        "person_detail.html",
        {
            "request": request,
            "person": person,
            "success": True
        }
    )
```

## Templates

### templates/base.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Random Person Generator{% endblock %}</title>
    <link rel="stylesheet" href="/static/style.css">
</head>
<body>
    <nav class="navbar">
        <div class="container">
            <div class="nav-brand">
                <a href="/">Random Person Generator</a>
            </div>
            <ul class="nav-menu">
                <li><a href="/">Home</a></li>
                <li><a href="/people">People List</a></li>
                <li><a href="/generate">Generate Person</a></li>
                <li><a href="/docs">API Docs</a></li>
            </ul>
        </div>
    </nav>

    <div class="container">
        {% block content %}{% endblock %}
    </div>

    <footer class="footer">
        <div class="container">
            <p>&copy; 2025 Random Person Generator. Built with FastAPI, 
            SQLAlchemy, and Jinja2.</p>
        </div>
    </footer>
</body>
</html>
```

### templates/index.html

```html
{% extends "base.html" %}

{% block title %}Home - Random Person Generator{% endblock %}

{% block content %}
<div class="hero">
    <h1>Welcome to Random Person Generator</h1>
    <p>Generate realistic random people with addresses and cars</p>
    
    <div class="action-buttons">
        <a href="/generate" class="btn btn-primary btn-lg">Generate New Person</a>
        <a href="/people" class="btn btn-secondary btn-lg">View All People</a>
        <a href="/docs" class="btn btn-info btn-lg">API Documentation</a>
    </div>
</div>

<div class="features">
    <h2>Features</h2>
    <div class="feature-grid">
        <div class="feature-card">
            <h3>🎲 Random Generation</h3>
            <p>Generate realistic random people with complete profiles</p>
        </div>
        <div class="feature-card">
            <h3>🏠 Addresses</h3>
            <p>Each person gets a randomly generated address</p>
        </div>
        <div class="feature-card">
            <h3>🚗 Vehicles</h3>
            <p>Assigned random vehicles with make, model, and year</p>
        </div>
        <div class="feature-card">
            <h3>🔌 RESTful API</h3>
            <p>Full REST API for programmatic access</p>
        </div>
    </div>
</div>
{% endblock %}
```

### templates/people_list.html

```html
{% extends "base.html" %}

{% block title %}People List - Random Person Generator{% endblock %}

{% block content %}
<div class="page-header">
    <h1>All People</h1>
    <a href="/generate" class="btn btn-primary">Generate New Person</a>
</div>

{% if people %}
    <div class="table-responsive">
        <table class="table">
            <thead>
                <tr>
                    <th>ID</th>
                    <th>Name</th>
                    <th>SSN</th>
                    <th>City</th>
                    <th>Car</th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody>
                {% for person in people %}
                <tr>
                    <td>#{{ person.id }}</td>
                    <td>{{ person.first_name }} {{ person.last_name }}</td>
                    <td>{{ person.ssn }}</td>
                    <td>
                        {% if person.address %}
                            {{ person.address.city }}, {{ person.address.state }}
                        {% else %}
                            N/A
                        {% endif %}
                    </td>
                    <td>
                        {% if person.car %}
                            {{ person.car.make }} {{ person.car.model }}
                        {% else %}
                            N/A
                        {% endif %}
                    </td>
                    <td>
                        <a href="/people/{{ person.id }}" class="btn btn-small">
                            View
                        </a>
                    </td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
    </div>

    <div class="pagination">
        {% if skip > 0 %}
            <a href="/people?skip={{ skip - limit }}&limit={{ limit }}" 
               class="btn btn-secondary">
                Previous
            </a>
        {% endif %}
        
        <span class="pagination-info">
            Showing {{ skip + 1 }} to {{ skip + people|length }}
        </span>
        
        {% if people|length == limit %}
            <a href="/people?skip={{ skip + limit }}&limit={{ limit }}" 
               class="btn btn-secondary">
                Next
            </a>
        {% endif %}
    </div>
{% else %}
    <div class="empty-state">
        <h2>No people found</h2>
        <p>Generate the first person to get started</p>
        <a href="/generate" class="btn btn-primary">Generate Person</a>
    </div>
{% endif %}
{% endblock %}
```

### templates/person_detail.html

```html
{% extends "base.html" %}

{% block title %}{{ person.first_name }} {{ person.last_name }} 
- Random Person Generator{% endblock %}

{% block content %}
<div class="page-header">
    <h1>{{ person.first_name }} {{ person.last_name }}</h1>
    <div class="actions">
        <a href="/people" class="btn btn-secondary">Back to List</a>
        <button class="btn btn-danger" onclick="deletePerson({{ person.id }})">
            Delete
        </button>
    </div>
</div>

{% if success %}
    <div class="alert alert-success">
        Person generated successfully!
    </div>
{% endif %}

<div class="detail-grid">
    <div class="detail-card">
        <h2>Personal Information</h2>
        <dl>
            <dt>ID:</dt>
            <dd>#{{ person.id }}</dd>
            <dt>First Name:</dt>
            <dd>{{ person.first_name }}</dd>
            <dt>Last Name:</dt>
            <dd>{{ person.last_name }}</dd>
            <dt>SSN:</dt>
            <dd>{{ person.ssn }}</dd>
        </dl>
    </div>

    {% if person.address %}
    <div class="detail-card">
        <h2>Address</h2>
        <dl>
            <dt>Street:</dt>
            <dd>{{ person.address.street }}</dd>
            <dt>City:</dt>
            <dd>{{ person.address.city }}</dd>
            <dt>State:</dt>
            <dd>{{ person.address.state }}</dd>
            <dt>Postal Code:</dt>
            <dd>{{ person.address.postal_code }}</dd>
            <dt>Country:</dt>
            <dd>{{ person.address.country }}</dd>
        </dl>
    </div>
    {% endif %}

    {% if person.car %}
    <div class="detail-card">
        <h2>Vehicle</h2>
        <dl>
            <dt>Make:</dt>
            <dd>{{ person.car.make }}</dd>
            <dt>Model:</dt>
            <dd>{{ person.car.model }}</dd>
            <dt>Color:</dt>
            <dd>
                <span class="color-badge" 
                      style="background-color: {{ person.car.color|lower }};">
                    {{ person.car.color }}
                </span>
            </dd>
            <dt>Year:</dt>
            <dd>{{ person.car.year }}</dd>
        </dl>
    </div>
    {% endif %}
</div>

<script>
async function deletePerson(personId) {
    if (confirm('Are you sure you want to delete this person?')) {
        try {
            const response = await fetch(`/api/people/${personId}`, {
                method: 'DELETE'
            });
            
            if (response.ok) {
                alert('Person deleted successfully');
                window.location.href = '/people';
            } else {
                alert('Failed to delete person');
            }
        } catch (error) {
            alert('Error: ' + error);
        }
    }
}
</script>
{% endblock %}
```

### templates/person_form.html

```html
{% extends "base.html" %}

{% block title %}Generate Person - Random Person Generator{% endblock %}

{% block content %}
<div class="page-header">
    <h1>Generate Random Person</h1>
</div>

<div class="form-container">
    <div class="form-card">
        <p class="form-description">
            Click the button below to generate a new random person with a 
            complete profile including address and vehicle.
        </p>
        
        <form method="POST" action="/generate">
            <div class="form-group">
                <button type="submit" class="btn btn-primary btn-lg">
                    Generate Random Person
                </button>
            </div>
        </form>
        
        <p class="form-note">
            This will create a new person entry in the database with randomly 
            generated personal information, address, and vehicle details.
        </p>
    </div>
    
    <div class="info-card">
        <h3>What Gets Generated?</h3>
        <ul>
            <li><strong>Name:</strong> Random first and last name</li>
            <li><strong>SSN:</strong> Random social security number</li>
            <li><strong>Address:</strong> Random street, city, state, postal code</li>
            <li><strong>Vehicle:</strong> Random make, model, color, and year (2010-2025)</li>
        </ul>
    </div>
</div>
{% endblock %}
```

## Static Styles (static/style.css)

```css
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background-color: #f5f5f5;
    color: #333;
    line-height: 1.6;
}

.container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 20px;
}

/* Navbar */
.navbar {
    background-color: #2c3e50;
    color: white;
    padding: 1rem 0;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.navbar .container {
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.nav-brand a {
    font-size: 1.5rem;
    font-weight: bold;
    color: white;
    text-decoration: none;
}

.nav-menu {
    display: flex;
    list-style: none;
    gap: 2rem;
}

.nav-menu a {
    color: white;
    text-decoration: none;
    transition: color 0.3s;
}

.nav-menu a:hover {
    color: #3498db;
}

/* Main Content */
.container {
    padding: 2rem 20px;
}

/* Buttons */
.btn {
    display: inline-block;
    padding: 0.5rem 1rem;
    border: none;
    border-radius: 4px;
    text-decoration: none;
    cursor: pointer;
    font-size: 1rem;
    transition: all 0.3s;
    text-align: center;
}

.btn-primary {
    background-color: #3498db;
    color: white;
}

.btn-primary:hover {
    background-color: #2980b9;
}

.btn-secondary {
    background-color: #95a5a6;
    color: white;
}

.btn-secondary:hover {
    background-color: #7f8c8d;
}

.btn-danger {
    background-color: #e74c3c;
    color: white;
}

.btn-danger:hover {
    background-color: #c0392b;
}

.btn-info {
    background-color: #9b59b6;
    color: white;
}

.btn-info:hover {
    background-color: #8e44ad;
}

.btn-small {
    padding: 0.3rem 0.6rem;
    font-size: 0.9rem;
}

.btn-lg {
    padding: 1rem 2rem;
    font-size: 1.1rem;
}

/* Hero Section */
.hero {
    text-align: center;
    padding: 3rem 0;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    border-radius: 8px;
    margin-bottom: 3rem;
}

.hero h1 {
    font-size: 2.5rem;
    margin-bottom: 1rem;
}

.hero p {
    font-size: 1.2rem;
    margin-bottom: 2rem;
}

.action-buttons {
    display: flex;
    gap: 1rem;
    justify-content: center;
    flex-wrap: wrap;
}

/* Features */
.features {
    margin: 3rem 0;
}

.feature-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: 2rem;
    margin-top: 2rem;
}

.feature-card {
    background: white;
    padding: 2rem;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    text-align: center;
}

.feature-card h3 {
    font-size: 1.3rem;
    margin-bottom: 1rem;
}

/* Page Header */
.page-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 2rem;
    padding-bottom: 1rem;
    border-bottom: 2px solid #ecf0f1;
}

.page-header h1 {
    margin: 0;
}

.actions {
    display: flex;
    gap: 1rem;
}

/* Tables */
.table-responsive {
    overflow-x: auto;
    margin-bottom: 2rem;
}

.table {
    width: 100%;
    border-collapse: collapse;
    background: white;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.table thead {
    background-color: #34495e;
    color: white;
}

.table th,
.table td {
    padding: 1rem;
    text-align: left;
    border-bottom: 1px solid #ecf0f1;
}

.table tbody tr:hover {
    background-color: #f9f9f9;
}

/* Forms */
.form-container {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 2rem;
    margin: 2rem 0;
}

.form-card,
.info-card {
    background: white;
    padding: 2rem;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.form-group {
    margin-bottom: 1.5rem;
}

.form-group label {
    display: block;
    margin-bottom: 0.5rem;
    font-weight: 500;
}

.form-group input,
.form-group select,
.form-group textarea {
    width: 100%;
    padding: 0.75rem;
    border: 1px solid #ddd;
    border-radius: 4px;
    font-size: 1rem;
}

.form-description {
    font-size: 1.1rem;
    margin-bottom: 1.5rem;
    color: #555;
}

.form-note {
    margin-top: 1rem;
    padding: 1rem;
    background-color: #ecf0f1;
    border-left: 4px solid #3498db;
    border-radius: 4px;
}

.info-card h3 {
    margin-bottom: 1rem;
}

.info-card ul {
    list-style: none;
    padding-left: 1rem;
}

.info-card li {
    margin-bottom: 0.75rem;
    padding-left: 1.5rem;
    position: relative;
}

.info-card li:before {
    content: "✓";
    position: absolute;
    left: 0;
    color: #27ae60;
    font-weight: bold;
}

/* Detail View */
.detail-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
    gap: 2rem;
    margin: 2rem 0;
}

.detail-card {
    background: white;
    padding: 1.5rem;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.detail-card h2 {
    margin-bottom: 1rem;
    color: #2c3e50;
}

.detail-card dl {
    display: grid;
    grid-template-columns: 150px 1fr;
    gap: 1rem;
}

.detail-card dt {
    font-weight: 600;
    color: #555;
}

.detail-card dd {
    color: #333;
}

.color-badge {
    display: inline-block;
    padding: 0.5rem 1rem;
    border-radius: 4px;
    color: white;
    font-weight: 500;
    text-shadow: 1px 1px 2px rgba(0,0,0,0.3);
}

/* Pagination */
.pagination {
    display: flex;
    gap: 1rem;
    justify-content: center;
    align-items: center;
    margin: 2rem 0;
}

.pagination-info {
    padding: 0.5rem 1rem;
}

/* Alerts */
.alert {
    padding: 1rem;
    border-radius: 4px;
    margin-bottom: 1rem;
}

.alert-success {
    background-color: #d4edda;
    color: #155724;
    border: 1px solid #c3e6cb;
}

/* Empty State */
.empty-state {
    text-align: center;
    padding: 3rem 2rem;
    background: white;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.empty-state h2 {
    color: #7f8c8d;
    margin-bottom: 1rem;
}

/* Footer */
.footer {
    background-color: #2c3e50;
    color: white;
    text-align: center;
    padding: 2rem 0;
    margin-top: 3rem;
    border-top: 1px solid #34495e;
}

/* Responsive */
@media (max-width: 768px) {
    .nav-menu {
        flex-direction: column;
        gap: 1rem;
    }
    
    .hero h1 {
        font-size: 1.8rem;
    }
    
    .page-header {
        flex-direction: column;
        align-items: flex-start;
        gap: 1rem;
    }
    
    .actions {
        width: 100%;
    }
    
    .form-container {
        grid-template-columns: 1fr;
    }
    
    .detail-card dl {
        grid-template-columns: 1fr;
    }
    
    .form-container {
        grid-template-columns: 1fr;
    }
}
```

## Updated Requirements (requirements.txt)

```
fastapi==0.104.1
uvicorn==0.24.0
sqlalchemy==2.0.23
faker==20.1.0
pydantic==2.5.0
jinja2==3.1.2
python-multipart==0.0.6
```

## How to Run

```bash
# Install dependencies
pip install -r requirements.txt

# Create the app directory structure
mkdir -p app/{models,schemas,repositories,controllers,services,templates,static}

# Run the application
python -m app.main
```

## Access Points

- **Web UI Home:** http://localhost:8000/
- **People List:** http://localhost:8000/people
- **Generate Person:** http://localhost:8000/generate
- **Person Details:** http://localhost:8000/people/{id}
- **REST API Docs:** http://localhost:8000/docs
- **ReDoc API Docs:** http://localhost:8000/redoc

This mirrors a C#/ASP.NET style where you have separate controllers for API endpoints and view endpoints, with Jinja2 templates rendering HTML responses alongside RESTful JSON responses!
