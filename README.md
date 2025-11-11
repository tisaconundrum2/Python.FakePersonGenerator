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
