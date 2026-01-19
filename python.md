# Python Errors

###
## Error Category: Error Category: OOP Inheritance / Attribute Override via super()

**Date:** 21 Apr 2025
**Context:** Subclass (RelatedEntityNotFoundError) was supposed to override a parent attribute (user_message) from RelationshipError.
Instead, the API response returned the default user_message from the parent class, despite clearly setting a new one in the child.

**Error:**
```python
Error: 
{
  "detail": "Related record does not exist"
}


Expected: 
{
  "detail": "Related Educator not found!"
}

```

**Code:**
```python
def resolve_fk_on_create():
    """
    Decorator to resolve foreign key errors during CREATE operations.
    Parses the error message to identify the specific foreign key constraint.
    """
    def decorator(func):
        @wraps(func)
        def wrapper(self, *args, **kwargs):
            try:
                return func(self, *args, **kwargs)
            except RelationshipError as e:
                context_obj = kwargs.get("data")
                if context_obj is None and len(args) > 1:
                    context_obj = args[1]

                resolved = FKResolver.resolve_fk_violation(
                    factory_class=self.__class__,
                    error_message=str(e),
                    context_obj=context_obj,
                    operation="create",
                    fk_map=fk_error_map
                )
                if resolved:
                    raise resolved
                raise e
        return wrapper
    return decorator







class RelationshipError(DBError):
    """Raised when a foreign key constraint is violated during data operations"""

    def __init__(self, error: str, operation: str, constraint: str | None = None):
        self.error = error
        self.constraint = constraint or "unknown"
        self.user_message = f"Related record does not exist"
        self.log_message = f"ForeignKeyViolation: during {operation} - {error}"
        super().__init__()

    def __str__(self):
        return self.error


class RelatedEntityNotFoundError(RelationshipError):
    """Raised when an entity cannot be found during attempted fk insertion"""
    def __init__(
            self, entity_model, identifier: UUID, display_name: str, operation: str,
            detail: str):

        self.user_message = f"Related {display_name} not found!"
        self.log_message = (f"Error during during fk insertion of {entity_model} with "
                            f"id:{identifier} during {operation}.")
        super().__init__(error=detail, operation=operation)

```


**Solution:**

```python
class RelationshipError(DBError):
    """Raised when a foreign key constraint is violated during data operations"""

    def __init__(self, error: str, operation: str, constraint: str | None = None):
        self.error = error
        self.constraint = constraint or "unknown"
        self.user_message = f"Related record does not exist"
        self.log_message = f"ForeignKeyViolation: during {operation} - {error}"
        super().__init__()

    def __str__(self):
        return self.error


class RelatedEntityNotFoundError(RelationshipError):
    """Raised when an entity cannot be found during attempted fk insertion"""
    def __init__(
            self, entity_model, identifier: UUID, display_name: str, operation: str,
            detail: str):
        super().__init__(error=detail, operation=operation)

        self.user_message = f"Related {display_name} not found!"
        self.log_message = (f"Error during during fk insertion of {entity_model} with "
                            f"id:{identifier} during {operation}.")
```

**Explanation:**  
The parent class RelationshipError sets a default user_message in its __init__.
The subclass set it's own self.user_message and then called super(), which made the parent parent attr overrde the child's.

**Lesson:** 
Always set or override subclass attributes after the super().__init__() call.

**Debug time:** 4 effin hours.

checked the decorator flow, error class, mapper and Middleware like a gazillion times. Literally everything except the inheritance chain

