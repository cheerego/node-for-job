## GraphQL

### 三个 Demo 学习 GraphQL
* Query
* Variable
* Alias
* Fragment
* ErrorFormat
* Mutation

### Demo1

```javascript
const express = require('express');
const express_graphql = require('express-graphql');
const {buildSchema} = require('graphql');

let schema = buildSchema(`
  type Query {
    hello: String
  }
`);

let root = {
    hello: () => {
        return 'hello world graphql'
    }
};
// Create an express server and a GraphQL endpoint
const app = express();
app.use('/graphql', express_graphql({
    schema: schema,
    rootValue: root,
    graphiql: true,//调试器 可以通过http://localhost:4000/graphql进入
    formatError: (err) => ({message: err.message, status: err.status}),
}));
app.listen(4000, () => console.log('Express GraphQL Server Now Running On http://localhost:4000/graphql'));
```


```
{
  hello
}
结果
{
  "data": {
    "hello": "hello world graphql"
  }
}
```


### Demo2
```javascript
const express = require('express');
const express_graphql = require('express-graphql');
const {buildSchema} = require('graphql');
// GraphQL schema
let schema = buildSchema(`
    type Query {
        course(id: Int!): Course
        courses(topic: String): [Course]
    },
    type Course {
        id: Int
        title: String
        author: String
        description: String
        topic: String
        url: String
    }
    
    type Mutation {
        updateCourseTopic(id: Int!, topic: String!): Course
    }
    
`);
let coursesData = [
    {
        id: 1,
        title: 'The Complete Node.js Developer Course',
        author: 'Andrew Mead, Rob Percival',
        description: 'Learn Node.js by building real-world applications with Node, Express, MongoDB, Mocha, and more!',
        topic: 'Node.js',
        url: 'https://codingthesmartway.com/courses/nodejs/'
    },
    {
        id: 2,
        title: 'Node.js, Express & MongoDB Dev to Deployment',
        author: 'Brad Traversy',
        description: 'Learn by example building & deploying real-world Node.js applications from absolute scratch',
        topic: 'Node.js',
        url: 'https://codingthesmartway.com/courses/nodejs-express-mongodb/'
    },
    {
        id: 3,
        title: 'JavaScript: Understanding The Weird Parts',
        author: 'Anthony Alicea',
        description: 'An advanced JavaScript course for everyone! Scope, closures, prototypes, this, build your own framework, and more.',
        topic: 'JavaScript',
        url: 'https://codingthesmartway.com/courses/understand-javascript/'
    }
]
let getCourse = function (args) {
    var id = args.id;
    return coursesData.filter(course => {
        return course.id == id;
    })[0];
}
let getCourses = function (args) {
    if (args.topic) {
        var topic = args.topic;
        return coursesData.filter(course => course.topic === topic);
    } else {
        return coursesData;
    }
}

let updateCourseTopic = function ({id, topic}) {
    coursesData = coursesData.map((item) => {
        if (item.id === id) {
            item.topic = topic;
        }
        return item;
    });
    return coursesData.find((item)=>{
        return item.id === id;
    })
};
let root = {
    course: getCourse,
    courses: getCourses,
    updateCourseTopic: updateCourseTopic
};
// Create an express server and a GraphQL endpoint
let app = express();
app.use('/graphql', express_graphql({
    schema: schema,
    rootValue: root,
    graphiql: true,
    formatError: (err) => ({message: err.message, status: err.status}),
}));
app.listen(4000, () => console.log('Express GraphQL Server Now Running On http://localhost:4000/graphql'));

```

#### 根据 CourseID 查询一个 Course
```
// 1
query getOneCourse($courseId:Int!){
  course(id:$courseId){
    id
    title
    author
    description
    url
    topic
  }
}

{
	"courseId": 1
}

结果
{
  "data": {
    "course": {
      "id": 1,
      "title": "The Complete Node.js Developer Course",
      "author": "Andrew Mead, Rob Percival",
      "description": "Learn Node.js by building real-world applications with Node, Express, MongoDB, Mocha, and more!",
      "url": "https://codingthesmartway.com/courses/nodejs/",
      "topic": "Node.js"
    }
  }
}

//2 且使用别名

{
  c:course(id:1){ 
    id
    title
    author
    description
    url
    topic
  }
}
结果
{
  "data": {
    "c": {
      "id": 1,
      "title": "The Complete Node.js Developer Course",
      "author": "Andrew Mead, Rob Percival",
      "description": "Learn Node.js by building real-world applications with Node, Express, MongoDB, Mocha, and more!",
      "url": "https://codingthesmartway.com/courses/nodejs/",
      "topic": "Node.js"
    }
  }
}

```
#### 根据 TopicName 查询多个 Courses
```
// 1
{
  courses(topic:"JavaScript"){
    id
    title
    author
    description
    url
    topic
  }
}
结果
{
  "data": {
    "courses": [
      {
        "id": 3,
        "title": "JavaScript: Understanding The Weird Parts",
        "author": "Anthony Alicea",
        "description": "An advanced JavaScript course for everyone! Scope, closures, prototypes, this, build your own framework, and more.",
        "url": "https://codingthesmartway.com/courses/understand-javascript/",
        "topic": "JavaScript"
      }
    ]
  }
}

// 2
query getCoursesByTopic($topicName:String!){
  courses(topic:$topicName){
    id
    title
    url
    author
    description
    topic
  }
}
{
	"topicName":  "Node.js"
}
结果
{
  "data": {
    "courses": [
      {
        "id": 1,
        "title": "The Complete Node.js Developer Course",
        "url": "https://codingthesmartway.com/courses/nodejs/",
        "author": "Andrew Mead, Rob Percival",
        "description": "Learn Node.js by building real-world applications with Node, Express, MongoDB, Mocha, and more!",
        "topic": "Node.js"
      },
      {
        "id": 2,
        "title": "Node.js, Express & MongoDB Dev to Deployment",
        "url": "https://codingthesmartway.com/courses/nodejs-express-mongodb/",
        "author": "Brad Traversy",
        "description": "Learn by example building & deploying real-world Node.js applications from absolute scratch",
        "topic": "Node.js"
      }
    ]
  }
}

```

#### 使用 Fragment
```
query getCoursesByTopic($topicName:String!){
  courses(topic:$topicName){
    ...CourseFields
  }
}

fragment CourseFields on Course {
  id
  title
  url
  author
  description
  topic
}

{
	"topicName":  "Node.js"
}

结果
{
  "data": {
    "courses": [
      {
        "id": 1,
        "title": "The Complete Node.js Developer Course",
        "url": "https://codingthesmartway.com/courses/nodejs/",
        "author": "Andrew Mead, Rob Percival",
        "description": "Learn Node.js by building real-world applications with Node, Express, MongoDB, Mocha, and more!",
        "topic": "Node.js"
      },
      {
        "id": 2,
        "title": "Node.js, Express & MongoDB Dev to Deployment",
        "url": "https://codingthesmartway.com/courses/nodejs-express-mongodb/",
        "author": "Brad Traversy",
        "description": "Learn by example building & deploying real-world Node.js applications from absolute scratch",
        "topic": "Node.js"
      }
    ]
  }
}

``` 

#### 使用 Mutation
```
mutation updateCourseTopic($id: Int!, $topic: String!) {
  updateCourseTopic(id: $id, topic: $topic) {
    ... CourseFields
  }
}

fragment CourseFields on Course {
  id
  title
  url
  author
  description
  topic
}
{
	"id": 1,
  "topic": "Golang"
}
结果
{
  "data": {
    "updateCourseTopic": {
      "id": 1,
      "title": "The Complete Node.js Developer Course",
      "url": "https://codingthesmartway.com/courses/nodejs/",
      "author": "Andrew Mead, Rob Percival",
      "description": "Learn Node.js by building real-world applications with Node, Express, MongoDB, Mocha, and more!",
      "topic": "Golang"
    }
  }
}

```

## Demo3
```javascript
var express = require('express');
var express_graphql = require('express-graphql');
var {buildSchema} = require('graphql');
const {find, filter} = require('lodash');
// GraphQL schema
var schema = buildSchema(`
   type Author {
    id: Int!
    firstName: String
    lastName: String
    posts: [Post] # the list of Posts by this author
  }

  type Post {
    id: Int!
    title: String
    author: Author
    votes: Int
  }

  # the schema allows the following query:
  type Query {
    posts: [Post]
    post(id:Int!):  Post
    author(id: Int!): Author
  }

  # this schema allows the following mutation:
  type Mutation {
    upvotePost (
      postId: Int!
    ): Post
  }
`);

const authors = [
    {id: 1, firstName: 'Tom', lastName: 'Coleman'},
    {id: 2, firstName: 'Sashko', lastName: 'Stubailo'},
    {id: 3, firstName: 'Mikhail', lastName: 'Novikov'},
];

const posts = [
    {id: 1, authorId: 1, title: 'Introduction to GraphQL', votes: 2},
    {id: 2, authorId: 2, title: 'Welcome to Apollo', votes: 3},
    {id: 3, authorId: 2, title: 'Advanced GraphQL', votes: 1},
    {id: 4, authorId: 3, title: 'Launchpad is Cool', votes: 7},
];

var root = {
    author: ({id}) => {
        return authors.find((author) => {
            return author.id === id;
        })
    },
    post: ({id}) => {
        let post = posts.find((post) => post.id === id)
        post.author = authors.find((item) => item.id === post.id);
        return post;
    },
    upvotePost: ({postId}) => {
        const post = posts.find((item) => item.id = postId);
        if (!post) {
            throw new Error(`Couldn't find post with id ${postId}`);
        }
        post.votes += 1;
        return post;
    },
};
// Create an express server and a GraphQL endpoint
var app = express();
app.use('/graphql', express_graphql({
    schema: schema,
    rootValue: root,
    graphiql: true,
    formatError: (err) => ({message: err.message, status: err.status}),
}));
app.listen(4000, () => console.log('Express GraphQL Server Now Running On http://localhost:4000/graphql'));

```

#### 级联查询
```
{
	post(id:1){
    id
    author {
      id
      firstName
    }
  }
}

{
  "data": {
    "post": {
      "id": 1,
      "author": {
        "id": 1,
        "firstName": "Tom"
      }
    }
  }
}
```

