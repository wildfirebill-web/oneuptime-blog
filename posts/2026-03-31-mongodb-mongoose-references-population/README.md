# How to Use Mongoose References and Population

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongoose, Population, Reference, Join

Description: Learn how to define references between Mongoose models and use populate() to perform cross-collection joins in MongoDB Node.js applications.

---

## References vs. Embedding in Mongoose

When related data lives in separate collections, store a reference (`ObjectId`) in one document that points to a document in another collection. Mongoose `populate()` resolves those references with additional queries - similar to a SQL JOIN, but done in the application layer.

## Defining a Reference Field

```javascript
const postSchema = new mongoose.Schema({
  title:   { type: String, required: true },
  body:    String,
  author:  { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  tags:    [{ type: mongoose.Schema.Types.ObjectId, ref: 'Tag' }]
});

const Post = mongoose.model('Post', postSchema);
```

The `ref` option tells Mongoose which model to use when populating the field.

## Basic populate()

```javascript
const post = await Post.findById(postId).populate('author');
console.log(post.author.username); // resolved User document
```

Without `populate`, `post.author` is just an ObjectId string. With it, Mongoose issues a second query to the `users` collection and replaces the ObjectId with the full document.

## Selecting Fields During Population

Reduce over-fetching by specifying which fields to include:

```javascript
const post = await Post.findById(postId)
  .populate('author', 'username email -_id')  // include only username and email
  .populate('tags', 'name');
```

The field selection string follows the same syntax as MongoDB projection.

## Populating Nested References

Populate references within populated documents:

```javascript
const post = await Post.findById(postId)
  .populate({
    path: 'author',
    populate: { path: 'company', select: 'name' }  // nested populate
  });

console.log(post.author.company.name);
```

Use deeply nested populate sparingly - each level adds a database round-trip.

## Populating in Query Results

```javascript
const posts = await Post.find({ published: true })
  .populate('author', 'username')
  .limit(20)
  .sort({ createdAt: -1 });
```

`populate()` works on all query methods: `find`, `findOne`, `findById`, etc.

## Model.populate() for Post-Query Population

Populate documents after they have already been retrieved:

```javascript
const posts = await Post.find().lean();
await Post.populate(posts, { path: 'author', select: 'username' });
```

Useful when you have an array of plain objects from a `.lean()` query.

## Filtering Populated Documents

Use the `match` option to filter which referenced documents are returned:

```javascript
const author = await User.findById(userId).populate({
  path: 'posts',
  match: { published: true },
  select: 'title createdAt',
  options: { sort: { createdAt: -1 }, limit: 5 }
});
```

Documents that do not match the filter are returned as `null` in arrays - filter them out with `.filter(Boolean)`.

## Virtual Population

Define a populate virtual for reverse references without storing the array on the parent:

```javascript
const userSchema = new mongoose.Schema({ username: String });

userSchema.virtual('posts', {
  ref:         'Post',
  localField:  '_id',
  foreignField: 'author'
});

const author = await User.findById(id).populate('posts');
```

## Summary

Mongoose `populate()` resolves ObjectId references across collections. Define references with `{ type: ObjectId, ref: 'ModelName' }`, then call `.populate('fieldName')` to replace IDs with documents. Select specific fields to avoid over-fetching, use nested populate for deeply related data, and prefer populate virtuals for reverse references to avoid maintaining arrays on both sides of a relationship.
