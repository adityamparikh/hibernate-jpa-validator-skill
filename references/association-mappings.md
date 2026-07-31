# Association Mappings

The four association types, their traps, and correct patterns. Applies to Hibernate 6 (JPA 3.1) and Hibernate 7 (Jakarta Persistence 3.2) alike unless a line says otherwise.

---

## @ManyToOne — The Workhorse

Child holds the FK. Always `fetch = FetchType.LAZY` and explicit `@JoinColumn(name = "...", nullable = ...)`.

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "author_id", nullable = false)
private Author author;
```

EAGER is JPA's default and is always wrong here — with 5 EAGER `@ManyToOne`s, every `findById` becomes a 5-way JOIN bomb regardless of what the caller needed.

---

## @OneToMany — Bidirectional with mappedBy

### Unidirectional @OneToMany: The Join Table Trap

```java
// ❌ WRONG — creates a post_comments join table with spurious DELETE+INSERT
@OneToMany
private List<Comment> comments = new ArrayList<>();
```

SQL when adding comment 3 to a post that already has comments 1 and 2:
```sql
DELETE FROM post_comments WHERE post_id = 1;
INSERT INTO post_comments (post_id, comment_id) VALUES (1, 1);  -- re-inserts old!
INSERT INTO post_comments (post_id, comment_id) VALUES (1, 2);  -- re-inserts old!
INSERT INTO post_comments (post_id, comment_id) VALUES (1, 3);  -- only new one needed
```

### Bidirectional: Always Use mappedBy

```java
// Parent side — inverse (no FK here)
@Entity
public class Post {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "post_seq")
    @SequenceGenerator(name = "post_seq", sequenceName = "post_id_seq", allocationSize = 50)
    private Long id;

    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Comment> comments = new ArrayList<>();

    // REQUIRED: sync helper methods for bidirectional consistency
    public void addComment(Comment comment) {
        comments.add(comment);
        comment.setPost(this);
    }

    public void removeComment(Comment comment) {
        comments.remove(comment);
        comment.setPost(null);
    }
}

// Child side — owns the FK
@Entity
public class Comment {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "comment_seq")
    @SequenceGenerator(name = "comment_seq", sequenceName = "comment_id_seq", allocationSize = 50)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id", nullable = false)
    private Post post;

    // equals/hashCode on business key, NOT id
}
```

Now only the necessary INSERT fires:
```sql
INSERT INTO comment (post_id, body) VALUES (1, 'Great post!');
```

### Cascade Rules for @OneToMany

```java
// Dependent children (comments, line items) — cascade all
@OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Comment> comments;

// Shared children (tags, categories) — NO cascade, NO orphanRemoval
@OneToMany(mappedBy = "post")
private List<PostTag> postTags;
```

**orphanRemoval = true** means: remove a Comment from the list → Hibernate DELETEs it. Only valid for entities that don't exist independently of the parent.

---

## @ManyToMany — Use Set, Not List

### The Bag Semantics Disaster

```java
// ❌ WRONG — List creates "bag" semantics
@ManyToMany
@JoinTable(name = "post_tag",
    joinColumns = @JoinColumn(name = "post_id"),
    inverseJoinColumns = @JoinColumn(name = "tag_id"))
private List<Tag> tags = new ArrayList<>();
```

When you add one tag to a post that already has 5 tags:
```sql
DELETE FROM post_tag WHERE post_id = 1;     -- deletes ALL 5 existing!
INSERT INTO post_tag (post_id, tag_id) VALUES (1, 1);  -- re-inserts all
INSERT INTO post_tag (post_id, tag_id) VALUES (1, 2);
INSERT INTO post_tag (post_id, tag_id) VALUES (1, 3);
INSERT INTO post_tag (post_id, tag_id) VALUES (1, 4);
INSERT INTO post_tag (post_id, tag_id) VALUES (1, 5);
INSERT INTO post_tag (post_id, tag_id) VALUES (1, 6);  -- only this was new!
```

This is because Hibernate cannot identify which element was added with a List bag — it nukes and rebuilds.

```java
// ✅ CORRECT — Set semantics: only INSERT the new row
@ManyToMany
@JoinTable(name = "post_tag",
    joinColumns = @JoinColumn(name = "post_id"),
    inverseJoinColumns = @JoinColumn(name = "tag_id"))
private Set<Tag> tags = new LinkedHashSet<>();  // LinkedHashSet for stable ordering
```

SQL for adding one tag:
```sql
INSERT INTO post_tag (post_id, tag_id) VALUES (1, 6);  -- only the new association
```

### Full Bidirectional @ManyToMany

```java
@Entity
public class Post {
    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(name = "post_tag",
        joinColumns = @JoinColumn(name = "post_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id"))
    private Set<Tag> tags = new LinkedHashSet<>();

    public void addTag(Tag tag) {
        tags.add(tag);
        tag.getPosts().add(this);
    }

    public void removeTag(Tag tag) {
        tags.remove(tag);
        tag.getPosts().remove(this);
    }
}

@Entity
public class Tag {
    @ManyToMany(mappedBy = "tags")
    private Set<Post> posts = new LinkedHashSet<>();
}
```

**Rules for @ManyToMany cascade:**
- Only `PERSIST` and `MERGE` — never `REMOVE` or `ALL` (would delete the shared Tag)
- Never `orphanRemoval = true`
- Tag's `equals`/`hashCode` must be based on business key (`name`), not ID

### When to Replace @ManyToMany with @OneToMany to Junction Entity

Promote the join table to a full entity when it carries attributes:

```java
// Instead of @ManyToMany between Post and Tag:
@Entity
@Table(name = "post_tag")
public class PostTag {
    @EmbeddedId
    private PostTagId id;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("postId")
    private Post post;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("tagId")
    private Tag tag;

    private Instant taggedAt;    // extra attribute → must be entity
    private String taggedBy;
}
```

---

## @OneToOne — Use @MapsId

The most misused association. Standard FK approach creates a separate column and requires an extra join. Use `@MapsId` to share the primary key.

```java
// ❌ STANDARD — separate FK column, extra join, EAGER by default
@Entity
public class User {
    @OneToOne(mappedBy = "user", fetch = FetchType.LAZY)
    private UserProfile profile;
}

@Entity
public class UserProfile {
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
}

// ✅ @MapsId — UserProfile.id == User.id, no extra column, no join needed
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
    @SequenceGenerator(name = "user_seq", sequenceName = "user_id_seq", allocationSize = 50)
    private Long id;

    @OneToOne(mappedBy = "user", fetch = FetchType.LAZY, cascade = CascadeType.ALL)
    private UserProfile profile;

    public void setProfile(UserProfile profile) {
        this.profile = profile;
        profile.setUser(this);
    }
}

@Entity
public class UserProfile {
    @Id
    private Long id;  // same value as user.id — no separate column

    @OneToOne(fetch = FetchType.LAZY)
    @MapsId
    @JoinColumn(name = "id")
    private User user;
}
```

**@MapsId benefits:**
- No extra FK column
- `entityManager.find(UserProfile.class, userId)` — no join needed
- Can load UserProfile directly by known user ID

---

## Bidirectional Sync Methods — Always Required

For every bidirectional association, add sync helpers on the **parent/owning side**:

```java
// Pattern: addChild / removeChild
public void addComment(Comment comment) {
    this.comments.add(comment);   // parent side
    comment.setPost(this);         // child side
}

public void removeComment(Comment comment) {
    this.comments.remove(comment); // parent side
    comment.setPost(null);          // child side
}
```

**Without sync methods:** First-level cache gets out of sync — `post.getComments()` returns stale list within same transaction, even though DB is correct.

---

## @OrderColumn

Use when order of a collection is significant and you need to persist it:

```java
@OneToMany(mappedBy = "lesson", cascade = CascadeType.ALL, orphanRemoval = true)
@OrderColumn(name = "position")
private List<Exercise> exercises = new ArrayList<>();
```

Hibernate maintains a `position` column on the child table. Reordering generates UPDATEs, not DELETE+INSERT. Works correctly with `List`, unlike bag semantics for relationships.

**Avoid when:** Collection is large and frequently reordered — each reorder UPDATEs N rows.
