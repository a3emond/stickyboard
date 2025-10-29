- # **StickyBoard – User Relations**

  ## **Overview**

  The **User Relations** system defines how friendships, connections, and blocking work across StickyBoard.
   Each record represents **one user’s directional relationship** with another.
   Mutual friendship exists when both users have corresponding `active` entries pointing to each other.

  This design enables simple queries, unilateral blocking, and precise control over social state, while maintaining a direct mapping between REST routes and the database’s composite key.

  ------

  ## **Database Schema**

  ```
  CREATE TYPE relation_status AS ENUM ('active','blocked','inactive');
  
  CREATE TABLE user_relations (
      user_id     uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
      friend_id   uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
      status      relation_status NOT NULL DEFAULT 'active',
      created_at  timestamptz NOT NULL DEFAULT now(),
      updated_at  timestamptz NOT NULL DEFAULT now(),
      UNIQUE (user_id, friend_id)
  );
  ```

  ### **Column Summary**

  | Column         | Type              | Description                                                  |
  | -------------- | ----------------- | ------------------------------------------------------------ |
  | **user_id**    | `uuid`            | The owner of this relation record.                           |
  | **friend_id**  | `uuid`            | The target user.                                             |
  | **status**     | `relation_status` | Current relationship state (`active`, `blocked`, or `inactive`). |
  | **created_at** | `timestamptz`     | Time when this relation entry was first created.             |
  | **updated_at** | `timestamptz`     | Automatically updated timestamp via trigger or application layer. |

  ### **Enum: `relation_status`**

  | Value        | Meaning                                            |
  | ------------ | -------------------------------------------------- |
  | **active**   | User considers the connection active.              |
  | **blocked**  | User has blocked this target (one-sided).          |
  | **inactive** | Soft-deleted or temporarily disabled relationship. |

  ------

  ## **Model Definition**

  ```
  [Table("user_relations")]
  public class UserRelation : IEntityUpdatable
  {
      [Key, Column("user_id", Order = 0)]
      public Guid UserId { get; set; }
  
      [Key, Column("friend_id", Order = 1)]
      public Guid FriendId { get; set; }
  
      [Column("status")]
      public RelationStatus Status { get; set; } = RelationStatus.active;
  
      [Column("created_at")]
      public DateTime CreatedAt { get; set; }
  
      [Column("updated_at")]
      public DateTime UpdatedAt { get; set; }
  }
  ```

  ------

  ## **Behavior**

  ### **1. Creation**

  When a friendship invite is accepted, two directional rows are created:

  | Direction | user_id | friend_id | status |
  | --------- | ------- | --------- | ------ |
  | A → B     | A       | B         | active |
  | B → A     | B       | A         | active |

  ### **2. Blocking**

  Blocking is unilateral:

  ```
  UPDATE user_relations
  SET status='blocked', updated_at=now()
  WHERE user_id=@blocker AND friend_id=@target;
  ```

  The other user’s entry remains unchanged.

  ### **3. Unfriending**

  To completely remove a friendship:

  ```
  DELETE FROM user_relations
  WHERE (user_id=@a AND friend_id=@b)
     OR (user_id=@b AND friend_id=@a);
  ```

  ### **4. Inactivation**

  Instead of deleting, both sides can be marked `inactive` for soft removal.

  ------

  ## **Query Patterns**

  ### **Get a User’s Relations**

  ```
  SELECT friend_id
  FROM user_relations
  WHERE user_id=@uid AND status='active';
  ```

  ### **Check Mutual Friendship**

  ```
  SELECT EXISTS (
    SELECT 1
    FROM user_relations a
    JOIN user_relations b
      ON a.user_id=@A AND a.friend_id=@B
     AND b.user_id=@B AND b.friend_id=@A
    WHERE a.status='active' AND b.status='active'
  );
  ```

  ### **List Blocked Users**

  ```
  SELECT friend_id
  FROM user_relations
  WHERE user_id=@uid AND status='blocked';
  ```

  ------

  ## **REST API Overview**

  ### **Routes**

  | Method   | Route                                    | Description                                               |
  | -------- | ---------------------------------------- | --------------------------------------------------------- |
  | `GET`    | `/api/userrelations`                     | Retrieve all active relations for the authenticated user. |
  | `POST`   | `/api/userrelations`                     | Create a new relation (adds both directions).             |
  | `PUT`    | `/api/userrelations/{userId}/{friendId}` | Update a specific directional relation by composite key.  |
  | `DELETE` | `/api/userrelations/{userId}/{friendId}` | Remove both directional entries for a pair.               |

  ### **Composite Key Semantics**

  All updates and deletions reference the **two-key combination** `(user_id, friend_id)` — exactly matching the database’s unique constraint.
   This ensures the API is **deterministic** and **schema-aligned**.

  ### **Security Enforcement**

  Controllers validate that the authenticated user matches the `userId` in the route:

  ```
  var callerId = User.GetUserId();
  if (callerId != userId)
      return Forbid();
  ```

  This guarantees users can only modify their own directional entries.

  ------

  ## **Service-Level Logic**

  ### **UserRelationService Overview**

  - **CreateAsync(a,b)** → inserts two reciprocal `active` entries.
  - **UpdateAsync(a,b,status)** → updates a single directional record.
  - **DeletePairAsync(a,b)** → deletes both directions.
  - **GetActiveAsync(uid)** → lists all active relations for a user.
  - **IsMutualAsync(a,b)** → confirms mutual `active` state.

  ------

  ## **Integration with Invites**

  When a user redeems a **friend invite**:

  1. The invite is marked as accepted.
  2. `UserRelationService.CreateAsync(senderId, newUserId)` executes.
  3. Two reciprocal `active` entries are created.
  4. Optionally, a system message (“You’re now connected”) is sent via `MessageService`.

  ------

  ## **Design Notes**

  | Principle                    | Description                                                  |
  | ---------------------------- | ------------------------------------------------------------ |
  | **Directional records**      | Each user manages their own relation entry.                  |
  | **Composite key addressing** | API routes map directly to `(user_id, friend_id)` keys.      |
  | **Independent blocking**     | Users can block without affecting the other side.            |
  | **Schema alignment**         | Routes and repository use the same composite key.            |
  | **Simple reads**             | Friend list = `SELECT * FROM user_relations WHERE user_id=@uid`. |
  | **Automatic timestamps**     | `updated_at` is maintained consistently.                     |

  ------

  ## **Summary**

  - Each user’s perspective of a relationship is stored as its own row.
  - Mutual friendships are represented by two `active` directional entries.
  - Blocking, inactivation, and deletion are unilateral.
  - API routes now use `(user_id, friend_id)` composite keys for perfect schema alignment.
  - The system integrates cleanly with invites and messaging, providing a simple yet flexible social layer for StickyBoard.