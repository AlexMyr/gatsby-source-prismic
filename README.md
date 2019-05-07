# gatsby-source-prismic

Source plugin for pulling data into [Gatsby][gatsby] from [prismic.io][prismic]
repositories.

## Table of Contents

- [gatsby-source-prismic](#gatsby-source-prismic)
  - [Table of Contents](#table-of-contents)
  - [Features](#features)
  - [Install](#install)
  - [How to use](#how-to-use)
  - [Providing JSON schemas](#providing-json-schemas)
  - [How to query](#how-to-query)
    - [Query Rich Text fields](#query-rich-text-fields)
    - [Query Link fields](#query-link-fields)
    - [Query Content Relation fields](#query-content-relation-fields)
    - [Query slices](#query-slices)
    - [Query direct API data as a fallback](#query-direct-api-data-as-a-fallback)
    - [Image processing](#image-processing)
- [Prismic Previews](#prismic-previews)
    - [usePrismicPreview()](#useprismicpreview)
    - [Return Value](#return-value)
    - [API](#api)
    - [Gotchas](#gotchas)
      - [Images](#images)
  - [Site's `gatsby-node.js` example](#sites-gatsby-nodejs-example)

## Features

- Supports Rich Text fields, slices, and content relation fields
- Supports `gatsby-transformer-sharp` and `gatsby-image` for image fields
- Utilizes `prismic-dom` to provide HTML and link values so you don't have to
  use `prismic-dom` directly
- Supports Prismic previews

## Install

```sh
npm install --save gatsby-source-prismic
```

## How to use

```js
// In your gatsby-config.js
plugins: [
  /*
   * Gatsby's data processing layer begins with “source”
   * plugins. Here the site sources its data from prismic.io.
   */
  {
    resolve: 'gatsby-source-prismic',
    options: {
      // The name of your prismic.io repository. This is required.
      // Example: 'gatsby-source-prismic-test-site' if your prismic.io address
      // is 'gatsby-source-prismic-test-site.prismic.io'.
      repositoryName: 'gatsby-source-prismic-test-site',

      // An API access token to your prismic.io repository. This is required.
      // You can generate an access token in the "API & Security" section of
      // your repository settings. Setting a "Callback URL" is not necessary.
      // The token will be listed under "Permanent access tokens".
      accessToken: 'example-wou7evoh0eexuf6chooz2jai2qui9pae4tieph1sei4deiboj',

      // Set a link resolver function used to process links in your content.
      // Fields with rich text formatting or links to internal content use this
      // function to generate the correct link URL.
      // The document node, field key (i.e. API ID), and field value are
      // provided to the function, as seen below. This allows you to use
      // different link resolver logic for each field if necessary.
      // See: https://prismic.io/docs/javascript/query-the-api/link-resolving
      linkResolver: ({ node, key, value }) => doc => {
        // Your link resolver
      },

      // Set a list of links to fetch and be made available in your link
      // resolver function.
      // See: https://prismic.io/docs/javascript/query-the-api/fetch-linked-document-fields
      fetchLinks: [
        // Your list of links
      ],

      // Set an HTML serializer function used to process formatted content.
      // Fields with rich text formatting use this function to generate the
      // correct HTML.
      // The document node, field key (i.e. API ID), and field value are
      // provided to the function, as seen below. This allows you to use
      // different HTML serializer logic for each field if necessary.
      // See: https://prismic.io/docs/nodejs/beyond-the-api/html-serializer
      htmlSerializer: ({ node, key, value }) => (
        type,
        element,
        content,
        children,
      ) => {
        // Your HTML serializer
      },

      // Provide an object of Prismic custom type JSON schemas to load into
      // Gatsby. Providing the schemas allows you to query for fields present in
      // your custom types even if they are unused in your documents.
      schemas: {
        // Your custom types mapped to schemas
      }

      // Set a default language when fetching documents. The default value is
      // '*' which will fetch all languages.
      // See: https://prismic.io/docs/javascript/query-the-api/query-by-language
      lang: '*',

      // Set a function to determine if images are downloaded locally and made
      // available for gatsby-transformer-sharp for use with gatsby-image.
      // The document node, field key (i.e. API ID), and field value are
      // provided to the function, as seen below. This allows you to use
      // different logic for each field if necessary.
      // This defaults to always return true.
      shouldNormalizeImage: ({ node, key, value }) => {
        // Return true to normalize the image or false to skip.
      },
    },
  },
]
```

## Providing JSON schemas

When querying for data in Gatsby via GraphQL, the list of available fields is
dependent on the data provided to Gatsby. By default, this plugin pulls all
data from Prismic, performs convenient transformations like HTML and link
conversions, and injects it directly into Gatsby.

For most cases, this works, but there are times when a field will exist in
Prismic, but not in Gatsby. You may come across this GraphQL error:

```
Cannot query field "my_field_id" on type "data_XX".
```

This happens when no documents in your Prismic repository uses the field being
queried.

To get around this, you can perform one of the following:

- **Recommended**: Provide the full JSON schema from Prismic to the source
  plugin via the `schemas` option in `gatsby-config.js`.

- Create a "placeholder" document for each custom type that has all values for
  all fields. In your `gatsby-node.js` file, set a way to find these documents
  and skip creating pages or delete them from the GraphQL store.

The recommended approach is to create a `schemas` directory in your project and
import them into your `gatsby-config.js` file.

```js
// In your gatsby-config.js
plugins: [
  {
    resolve: 'gatsby-source-prismic',
    options: {
      // ...
      schemas: {
        page: require('./src/schemas/page.json'),
        blogPost: require('./src/schemas/blogPost.json'),
      },
      // ...
    },
  },
]
```

Each schema file should be populated with the contents of the "JSON editor" tab
in the Prismic Custom Type editor.

See the official docs for more details: [How to version custom
types][prismic-version-custom-types].

## How to query

You can query nodes created from Prismic using GraphQL like the following:

**Note**: Learn to use the GraphQL tool and Ctrl+Spacebar at
<http://localhost:8000/___graphql> to discover the types and properties of your
GraphQL model.

```graphql
{
  allPrismicPage {
    edges {
      node {
        id
        uid
        first_publication_date
        last_publication_date
        data {
          title {
            text
          }
          content {
            html
          }
        }
      }
    }
  }
}
```

All documents are pulled from your repository and created as
`prismic${contentTypeName}` and `allPrismic${contentTypeName}`, where
`${contentTypeName}` is the API ID of your document's content type.

For example, if you have `Product` as one of your content types, you will be
able to query it like the following:

```graphql
{
  allPrismicProduct {
    edges {
      node {
        id
        data {
          name
          price
          description {
            html
          }
        }
      }
    }
  }
}
```

### Query Rich Text fields

Data from fields with rich text formatting (e.g. headings, bold, italic) is
transformed to provide HTML and text versions. This uses the official
[prismic-dom][prismic-dom] library and the `linkResolver` and `htmlSerializer`
functions from your site's `gatsby-node.js` to create the HTML and text fields.

**Note**: If you need to access the raw data, the original data is accessible
using the `raw` field, though use of this field is discouraged.

```graphql
{
  allPrismicPage {
    edges {
      node {
        id
        data {
          title {
            text
          }
          content {
            html
          }
        }
      }
    }
  }
}
```

### Query Link fields

Link fields are processed using the official [prismic-dom][prismic-dom] library
and the `linkResolver` function from your site's `gatsby-node.js`. The resolved
URL is provided at the `url` field.

If the link type is a web link (i.e. a URL external from your site), the URL is
provided without additional processing.

All other URL fields, such as `target`, `lang`, and `isBroken`, are provided on
the field, as well.

The `target` field defaults to an empty string. This allows you to always query
the `target` field even if it is not set in Prismic.

**Note**: If you need to access the raw data, the original data is accessible
using the `raw` field, though use of this field is discouraged.

```graphql
{
  allPrismicPage {
    edges {
      node {
        id
        data {
          featured_post {
            url
            target
          }
        }
      }
    }
  }
}
```

### Query Content Relation fields

Content Relation fields relate a field to another document. Since fields from
within the related document is often needed, the document data is provided at
the `document` field. The resolved URL to the document using the official
[prismic-dom][prismic-dom] library and the `linkResolver` function from your
site's `gatsby-node.js` is also provided at the `url` field.

**Note**: Data within the `document` field is wrapped in an array. Due to the
method in which Gatsby processes one-to-one node relationships, this
work-around is necessary to ensure the field can accommodate different content
types. This may be fixed in a later Gatsby relase.

Querying data on the `document` field is handled the same as querying slices.
Please read the [Query slices](#query-slices) section for details.

**Note**: If you need to access the raw data, the original data is accessible
using the `raw` field, though use of this field is discouraged.

```graphql
{
  allPrismicPage {
    edges {
      node {
        id
        data {
          featured_post {
            url
            document {
              __typename
              ... on PrismicPost {
                data {
                  title {
                    text
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

### Query slices

Prismic slices allow you to build a flexible series of content blocks. Since
the content structure is dynamic, querying the content is handled differently
than other fields.

To access slice fields, you need to use GraphQL [inline
fragments][graphql-inline-fragments]. This requires you to know types of nodes.
The easiest way to get the type of nodes is to use the `/___graphql` debugger
and run the below query (adjust the document type and field name).

```graphql
{
  allPrismicPage {
    edges {
      node {
        id
        data {
          body {
            __typename
          }
        }
      }
    }
  }
}
```

When you have node type names, you can use them to create inline fragments.

Full example:

```graphql
{
  allPrismicPage {
    edges {
      node {
        id
        data {
          body {
            __typename
            ... on PrismicPageBodyRichText {
              text {
                html
              }
            }
            ... on PrismicPageBodyQuote {
              quote {
                html
              }
              credit {
                text
              }
            }
            ... on PrismicPageBodyFootnote {
              content {
                html
              }
            }
          }
        }
      }
    }
  }
}
```

### Query direct API data as a fallback

If you find you cannot query the data you need through the GraphQL interface,
you can get the raw response from the [prismic-javascript][prismic-javascript]
API using the `dataString` field.

This field contains the whole node's original data before processing as a
string generated using `JSON.stringify`.

This is absolutely discouraged as it defeats the purpose of Gatsby's GraphQL
data interface, but it is available if necessary

```graphql
{
  allPrismicPage {
    edges {
      node {
        id
        dataString
      }
    }
  }
}
```

### Image processing

To use image processing you need `gatsby-transformer-sharp`,
`gatsby-plugin-sharp`, and their dependencies `gatsby-image` and
`gatsby-source-filesystem` in your `gatsby-config.js`.

You can apply image processing to any image field on a document. Image
processing of inline images added to Rich Text fields is currently not
supported.

To access image processing in your queries, you need to use this pattern, where
`...ImageFragment` is one of the [`gatsby-transformer-sharp`
fragments][gatsby-image-fragments]:

```graphql
{
  allPrismicPage {
    edges {
      node {
        id
        data {
          imageFieldName {
            localFile {
              childImageSharp {
                ...ImageFragment
              }
            }
          }
        }
      }
    }
  }
}
```

Full example:

```graphql
{
  allPrismicPage {
    edges {
      node {
        id
        data {
          imageFieldName {
            localFile {
              childImageSharp {
                resolutions(width: 500, height: 300) {
                  ...GatsbyImageSharpResolutions_withWebp
                }
              }
            }
          }
        }
      }
    }
  }
}
```

To learn more about image processing, check the documentation of
[gatsby-plugin-sharp][gatsby-plugin-sharp].

# Prismic Previews

### usePrismicPreview()

The `usePrismicPreview()` React hook allows for querying and normalizing
responses from Prismic's API. An example is shown below:

```jsx
import { usePrismicPreview } from 'gatsby-source-prismic'

const PreviewPage = ({ location }) => {
  const { previewData } = usePrismicPreview({
    location,
    linkResolver: doc => doc.uid,
    fetchLinks: ['page.parent'],
    repositoryName: process.env.GATSBY_PRISMIC_REPOSITORY_NAME,
    accessToken: process.env.GATSBY_PRISMIC_ACCESS_TOKEN,
  })

  return previewData ? <Spinner /> : <PageTemplate data={previewData} />
}
```

### Return Value

Returns an object with the following keys:

- `previewData`: An object with the same key-value shape that `gatsby-source-prismic` generates at build time, so it can be provided to templates & pages directly.
- `path`: A string of the resolved path for the previewed Prismic doc. This is determined via the provided `linkResolver`.

### API

Accepts the following parameters via an object:

- `location`: **Required**. The location object from `@reach/router`. This is used to read the preview token and doc ID from Prismic to send a preview request.
- `linkResolver`: **Required**. `usePrismicPreview()` uses this link resolver function to determine the `path` of the previewed doc.
- `htmlSerializer`: Function that maps rich text fields to HTML. Should be the same function provided to the plugin configuration.
- `fetchLinks`: A list of links to fetch for the previewed document.
- `repositoryName`: Your Prismic repository name.
- `accessToken`: Your prismic access token.

> ⚠️ Since preview API requests are made in the browser, your access token will be exposed to the client.

### Gotchas

#### Images 

Since data normalization happens at run-time, we cannot perform the same image optimizations that we do at build-time. Instead, `usePrismicPreview()` returns the `url` field for an image.

> ⚛️ A smart image component that conditionally uses `url` or `gatsby-image` data is recommended for preview parity.

## Site's `gatsby-node.js` example

```jsx
const path = require('path')

exports.createPages = async ({ graphql, actions }) => {
  const { createPage } = actions

  const pages = await graphql(`
    {
      allPrismicPage {
        edges {
          node {
            id
            uid
            template
          }
        }
      }
    }
  `)

  const pageTemplates = {
    Light: path.resolve('./src/templates/light.js'),
    Dark: path.resolve('./src/templates/dark.js'),
  }

  pages.data.allPrismicPage.edges.forEach(edge => {
    createPage({
      path: `/${edge.node.uid}`,
      component: pageTemplates[edge.node.template],
      context: {
        id: edge.node.id,
      },
    })
  })
}
```

[gatsby]: https://www.gatsbyjs.org/
[prismic]: https://prismic.io/
[prismic-dom]: https://github.com/prismicio/prismic-dom
[prismic-javascript]: https://github.com/prismicio/prismic-javascript
[prismic-version-custom-types]: https://user-guides.prismic.io/content-modeling-and-custom-types/version-and-changes-of-custom-types/how-to-version-custom-types
[graphql-inline-fragments]: http://graphql.org/learn/queries/#inline-fragments
[gatsby-plugin-sharp]: https://github.com/gatsbyjs/gatsby/blob/master/packages/gatsby-plugin-sharp
[gatsby-image-fragments]: https://github.com/gatsbyjs/gatsby/blob/master/packages/gatsby-image#gatsby-transformer-sharp
