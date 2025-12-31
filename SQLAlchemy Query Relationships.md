# SQLAlchemy Query Relationships - Database-Level Joins

Yes! SQLAlchemy provides several ways to specify joins that execute at the database level without loading everything into memory. Here are the main approaches:

## 1. **Using `joinedload()` - Eager Loading with Single Query**

```python
from sqlalchemy.orm import joinedload
from sqlalchemy import select

# Returns a single query with JOINs, not multiple queries
person = db.query(Person).options(
    joinedload(Person.address),
    joinedload(Person.car)
).filter(Person.id == 1).first()

# Or with SQLAlchemy 2.0 style (select):
stmt = select(Person).options(
    joinedload(Person.address),
    joinedload(Person.car)
).where(Person.id == 1)

person = db.execute(stmt).unique().scalar_one_or_none()
```

## 2. **Using `selectinload()` - Multiple Queries but Optimized**

```python
from sqlalchemy.orm import selectinload

# Executes 2-3 queries but is often more efficient than joinedload
people = db.query(Person).options(
    selectinload(Person.address),
    selectinload(Person.car)
).all()

# SQLAlchemy 2.0 style:
stmt = select(Person).options(
    selectinload(Person.address),
    selectinload(Person.car)
)

people = db.execute(stmt).unique().scalars().all()
```

## 3. **Using `contains_eager()` - Join with Filtering**

```python
from sqlalchemy.orm import contains_eager
from sqlalchemy import join

# Use when you want to filter by related table
people = db.query(Person).join(Address).options(
    contains_eager(Person.address)
).filter(Address.city == "New York").all()

# SQLAlchemy 2.0 style:
stmt = select(Person).join(Address).options(
    contains_eager(Person.address)
).where(Address.city == "New York")

people = db.execute(stmt).unique().scalars().all()
```

## 4. **Using Explicit `join()` - Full Control**

```python
# Execute a single query with explicit JOINs
people = db.query(Person, Address, Car).join(
    Address, Person.address_id == Address.id
).join(
    Car, Person.car_id == Car.id
).filter(Address.city == "New York").all()

# Returns tuples: [(Person, Address, Car), ...]
```

## Updated Services with Efficient Queries

```python
import random
from faker import Faker
from typing import List, Optional
from sqlalchemy.orm import Session, joinedload, selectinload
from sqlalchemy import select

from app.repositories.repository import (
    PersonRepository, 
    AddressRepository, 
    CarRepository
)
from app.models.models import Person, Address, Car
from app.schemas.schemas import PersonCreate, AddressCreate, CarCreate

fake = Faker()

class PersonService:
    def __init__(self):
        self.person_repository = PersonRepository()
        self.address_repository = AddressRepository()
        self.car_repository = CarRepository()
    
    def create_random_person(self, db: Session):
        """Create a random person with address and car"""
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
        car_makes = ["Toyota", "Honda", "Ford", "Chevrolet", "BMW", 
                     "Tesla", "Audi"]
        car_models = {
            "Toyota": ["Camry", "Corolla", "RAV4", "Highlander"],
            "Honda": ["Civic", "Accord", "CR-V", "Pilot"],
            "Ford": ["F-150", "Mustang", "Explorer", "Escape"],
            "Chevrolet": ["Silverado", "Malibu", "Equinox", "Tahoe"],
            "BMW": ["3 Series", "5 Series", "X3", "X5"],
            "Tesla": ["Model 3", "Model Y", "Model S", "Model X"],
            "Audi": ["A4", "A6", "Q5", "Q7"]
        }
        car_colors = ["Red", "Blue", "Black", "White", "Silver", 
                      "Green", "Yellow"]
        
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
        """
        Get a single person with all relationships.
        Executes ONE query with JOINs at the database level.
        """
        return db.query(Person).options(
            joinedload(Person.address),
            joinedload(Person.car)
        ).filter(Person.id == person_id).first()
    
    def get_people(self, db: Session, skip: int = 0, limit: int = 100):
        """
        Get all people with relationships efficiently.
        Uses selectinload for optimal query plan.
        """
        return db.query(Person).options(
            selectinload(Person.address),
            selectinload(Person.car)
        ).offset(skip).limit(limit).all()
    
    def get_people_in_city(self, db: Session, city: str, 
                           skip: int = 0, limit: int = 100):
        """
        Get people filtered by city using contains_eager.
        Filters at database level, not in memory.
        """
        return db.query(Person).join(Address).options(
            contains_eager(Person.address)
        ).filter(
            Address.city == city
        ).offset(skip).limit(limit).all()
    
    def get_people_with_car(self, db: Session, car_make: str,
                            skip: int = 0, limit: int = 100):
        """
        Get people by car make using contains_eager.
        """
        return db.query(Person).join(Car).options(
            contains_eager(Person.car)
        ).filter(
            Car.make == car_make
        ).offset(skip).limit(limit).all()
    
    def delete_person(self, db: Session, person_id: int):
        """Delete a person and associated address and car"""
        person = self.get_person(db, person_id)
        
        address_id = person.address_id
        car_id = person.car_id
        
        self.person_repository.delete(db, person_id)
        self.address_repository.delete(db, address_id)
        self.car_repository.delete(db, car_id)
        
        return person
```

## SQLAlchemy 2.0 Style (More Modern)

```python
from sqlalchemy import select
from sqlalchemy.orm import Session, joinedload, selectinload

class PersonService:
    def get_person_v2(self, db: Session, person_id: int):
        """SQLAlchemy 2.0 style - Single query with joins"""
        stmt = select(Person).options(
            joinedload(Person.address),
            joinedload(Person.car)
        ).where(Person.id == person_id)
        
        return db.execute(stmt).unique().scalar_one_or_none()
    
    def get_people_v2(self, db: Session, skip: int = 0, limit: int = 100):
        """SQLAlchemy 2.0 style - Multiple optimized queries"""
        stmt = select(Person).options(
            selectinload(Person.address),
            selectinload(Person.car)
        ).offset(skip).limit(limit)
        
        return db.execute(stmt).unique().scalars().all()
    
    def get_people_by_city_v2(self, db: Session, city: str):
        """SQLAlchemy 2.0 style - Filtered join"""
        stmt = select(Person).join(Address).options(
            contains_eager(Person.address)
        ).where(Address.city == city)
        
        return db.execute(stmt).unique().scalars().all()
```

## Comparison of Approaches

```python
# ❌ BAD - N+1 Query Problem (loads person, then queries for each address/car)
people = db.query(Person).all()
for person in people:
    print(person.address.city)  # Triggers a query for EACH person

# ✅ GOOD - joinedload (1 query with LEFT OUTER JOINs)
people = db.query(Person).options(
    joinedload(Person.address),
    joinedload(Person.car)
).all()

# ✅ GOOD - selectinload (2-3 queries but more efficient for 1-to-many)
people = db.query(Person).options(
    selectinload(Person.address),
    selectinload(Person.car)
).all()

# ✅ GOOD - contains_eager (1 query, filtered at DB level)
people = db.query(Person).join(Address).options(
    contains_eager(Person.address)
).filter(Address.city == "New York").all()
```

## Key Differences

| Strategy | Queries | Best For | Notes |
|----------|---------|----------|-------|
| `joinedload()` | 1 | 1-to-1 relationships | Uses LEFT OUTER JOIN |
| `selectinload()` | 2-3 | 1-to-many relationships | Avoids cartesian products |
| `contains_eager()` | 1 | Filtering by related table | Must use `.join()` |
| Explicit `join()` | 1 | Complex queries | Full control, returns tuples |

All of these execute **at the database level** - no in-memory manipulation!
