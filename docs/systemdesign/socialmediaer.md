Design a ER diagram for a social media platform managing users, posts, comments, and likes

## **Table attributes**

- User

Represents people using the platform.

Attributes: user_id(PK), username, email, password, created_at, bio

- Post

Content created by users.

Attributes: post_id (PK), user_id (FK) → User, content, media_url, created_at

Relationship: One User creates many Posts

- Comment

Users can comment on posts.

Attributes: comment_id (PK), post_id (FK) → Post, user_id (FK) → User, comment_text, created_at

Relationship:One Post has many Comments, One User writes many Comments

- Like

Users can like posts.

Attributes: like_id (PK), user_id (FK) → User, post_id (FK) → Post, created_at

Relationship: One User can like many Posts, One Post can have many Likes

## **Relationship Summary**

Entity	Relationship	Entity
User	creates	Post
User	writes	Comment
Post	has	Comment
User	likes	Post

## **ER Diagram (Conceptual)**

        +-------------+
        |   USERS     |
        +-------------+
        | user_id PK  |
        | username    |
        | email       |
        | password    |
        | created_at  |
        +-------------+
              |
              | 1
              |
              | N
        +-------------+
        |    POSTS    |
        +-------------+
        | post_id PK  |
        | user_id FK  |
        | content     |
        | media_url   |
        | created_at  |
        +-------------+
          |        |
        1 |        | N
          |        |
      +---------+  +-----------+
      | COMMENTS|  |   LIKES   |
      +---------+  +-----------+
      |comment_id| | like_id   |
      |post_id FK| | user_id FK|
      |user_id FK| | post_id FK|
      |text      | | created_at|
      |created_at| +-----------+
      +---------+

## **Cardinality**

Relationship : Type

User → Post	1 : Many

Post → Comment	1 : Many

User → Comment	1 : Many

User → Post (Likes)	Many : Many

(Many-to-many resolved using Like table)