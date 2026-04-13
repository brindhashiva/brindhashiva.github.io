Title: Python Variables, Lists, and Dictionaries
Date: 2026-01-01
Category: Programming
Tags: Python, Variables, Lists, Dictionary, Beginner
Slug: 
Status: 

Have you ever wondered how Python programs store and manage data efficiently without creating too many variables? Three fundamental concepts make this possible: variables, lists, and dictionaries.

## Variables in Python

**Variable** — A variable is used to store data. It acts like a container that holds a value which can be reused later in the program.

**Types of Variables** — Python supports multiple data types like integer, float, string, and boolean.

```python
x = 10          # Integer
y = 3.14        # Float
text = "Hello"  # String
is_valid = True # Boolean
```

## Lists in Python

**List** — A list is a collection of multiple values stored in a single variable. It is ordered and mutable.

**List Operations** — Lists support operations like adding, removing, slicing, and looping.

```python

items = ["pen", "pencil", "eraser"]

print(items[0])       # Indexing
print(items[1:3])     # Slicing

items.append("scale") # Add
items.pop()           # Remove

for item in items:
    print(item)
```

## Dictionaries in Python

**Dictionary** — A dictionary stores data in key-value pairs. Each key is unique and used to access its value.

**Dictionary Operations** — You can add, update, remove, and access data using keys.

```python
student = {
    "name": "Brindha",
    "age": 20
}

print(student["name"])     # Access
print(student.get("age"))  # Safe access

student["age"] = 21        # Update
student["course"] = "AI & DS"

student.pop("age")         # Remove
```

## Why These Concepts Matter
- Variables store single values
- Lists store multiple ordered values
- Dictionaries store key-value data

These three concepts form the foundation of Python programming.