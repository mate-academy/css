# Mate Frontend Style Guide <!-- omit in toc -->

[go/mate-frontend-style-guide](http://go/mate-frontend-style-guide)

- [1. General](#1-general)
    - [1.1. Always add NoSSR to AuthUser query on landings](#11-always-add-nossr-to-authuser-query-on-landings)
    - [1.2. Rename old components before refactoring](#12-rename-old-components-before-refactoring)
    - [1.3. Create new components for A/B tests](#13-create-new-components-for-ab-tests)
    - [1.4. Do not throw unhandled exceptions](#14-do-not-throw-unhandled-exceptions)
- [2. CSS](#2-css)
    - [2.1. Use `rem-calc` function for "size" and "indent" values instead of hardcoding pixels](#21-use-rem-calc-function-for-size-and-indent-values-instead-of-hardcoding-pixels)
    - [2.2. Use `em` units for font-related properties](#22-use-em-units-for-font-related-properties)
    - [2.3. Prefer internal atomic styles for indents](#23-prefer-internal-atomic-styles-for-indents)
- [3. UI kit](#3-ui-kit)
    - [3.1 Follow process of adding new UI elements](#31-follow-process-of-adding-new-ui-elements)
    - [3.2 DO NOT EXTEND visuals of existing components by adding props like `isLarge`, `isRounded`, `isSomethingElse`](#32-do-not-extend-visuals-of-existing-components-by-adding-props-like-islarge-isrounded-issomethingelse)
    - [3.3 Always add new components to the UI kit. Follow figma naming](#33-always-add-new-components-to-the-ui-kit-follow-figma-naming)
    - [3.4 Document props using js-doc comments](#34-document-props-using-js-doc-comments)
- [4. Raster assets images](#4-raster-assets-images)
    - [4.1 Follow guide when adding raster images to the website](#41-follow-guide-when-adding-raster-images-to-the-website)
    - [4.2 Emojis naming](#42-emojis-naming)
        - [4.2.1 Follow DRY](#421-follow-dry)
        - [4.2.2 Naming rules](#422-naming-rules)

## 1. General
#### 1.1. Always add NoSSR to AuthUser query on landings
**Check how it works and cacheable pages in `frontend/ssr-cache`**

>❓Why? Landing pages are cached on the server side to speed up content delivery to the end user. Basically, the rendered HTML is stored and SSR is not happenning afterwards. Executing `AuthUser` query in such conditions leads to bugs including the most critical one with shared user data.

1. Cache is empty, random user Bob opens landing
2. Page is generated with Bob's `AuthUser` in the context, HTML is stored in cache
3. Every new visitor receives Bob's HTML including personal data

```js
// ❌ bad
const { data } = useAuthUserQuery();

// ✅ good
const { data } = useAuthUserQuery({
  ssr: false;
});
```

#### 1.2. Rename old components & functions before refactoring
Let's say the `useIsOpenState` component needs refactoring and it's quite complicated to make changes in place. Before creating new `useIsOpenState`, rename the existing one to `useIsOpenStateDeprecated`. It works extremely good for cleanup afterwards.

When refactoring a widely-used component, it’s better to mark the old version as deprecated, rather than refactoring all usages and removing it immediately. Use a deprecation annotation in the old component with an explanation of the changes and how to migrate. This allows developers to see the deprecation notice in their IDE and understand how to update their code accordingly.

Why not naming new component with `New` suffix? Because it will be hard to understand what is deprecated and another renaming step will be required afterwards (`useIsOpenStateNew -> useIsOpenState`)

```jsx
// ❌ bad
// old component - hooks/useIsOpenState.tsx
export const useIsOpenState = () => // ...

// new component - hooks/useIsOpenStateNew.tsx
export const useIsOpenStateNew = () => // ...

// ✅ good
// old component - hooks/useIsOpenStateDeprecated.tsx
/**
 * @deprecated Use useIsOpenState instead.The type of return value was changed from array to object.
 */
export const useIsOpenStateDeprecated = () => // ...

// new component - hooks/useIsOpenState.tsx
export const useIsOpenState = () => // ...
```

#### 1.3. Create new components for A/B tests
When we run A/B tests, there's often a need to create new components. It's better to create new components instead of adding new props to existing ones. It will be easier to clean up afterward

But how to handle naming in this case? Adding the `Deprecated` suffix doesn't seem right because the component doesn't become deprecated. The best solution is to add `V1`, `V2`, and so on suffixes to a new component, where `V[number]` stands for the variant of the test

```jsx
// ❌ bad
// old component - components/Chat/Chat.tsx
export const Chat: FC<Props> // ...

// new component - components/ChatNew/ChatNew.tsx
export const ChatNew: FC<Props> // ...

// ❌ bad
// old component - components/ChatDeprecated/ChatDeprecated.tsx
export const ChatDeprecated: FC<Props> // ...

// new component - components/Chat/Chat.tsx
export const Chat: FC<Props> // ...

// ✅ good
// old component - components/Chat/Chat.tsx
export const Chat: FC<Props> // ...

// new component - components/ChatV[1,2,3]/ChatV[1,2,3].tsx
export const ChatV1: FC<Props> // ...
```

#### 1.4. Do not throw unhandled exceptions
We avoid showing "white screen" to the user that occurs as a result of unhandled exception. We prefer logging an error to AWS CloudWatch and showing a user-friendly error message instead. Also it is possible to use try-catch block to handle exceptions in a proper way.

```tsx
// ❌ bad
const getChatNameByType = (type: ChatType): string => {
  switch (type) {
    case ChatType.Private:
      return 'Private chat';
    case ChatType.Public:
      return 'Public chat';
    default:
      throw new Error('Unknown chat type');
  }
}

// ✅ good
const getChatNameByType = (type: ChatType): string => {
  switch (type) {
    case ChatType.Private:
      return 'Private chat';
    case ChatType.Public:
      return 'Public chat';
    default:
      logger.error(`Unknown chat type: ${type}`);

      return 'Unrecognized chat';
  }
}
```
```tsx
// ❌ bad
const getTooltipPosition = (size: number): Position => {
  if (hasEnoughSpaceAbove(size)) {
    return Position.Top;
  }

  if (hasEnoughSpaceBelow(size)) {
    return Position.Bottom;
  }

  throw new Error('No space for tooltip');
}

// ✅ good
const getTooltipPosition = (size: number): Position => {
  if (hasEnoughSpaceAbove(size)) {
    return Position.Top;
  }

  if (hasEnoughSpaceBelow(size)) {
    return Position.Bottom;
  }

  logger.error('No space for tooltip');

  // Yes, it will not be fit in the viewport.
  // But it's still better than white screen.
  return Position.Top;
}
```
```tsx
// ❌ bad
const getMessageJSX = (rawJSX: string): JSX.Element => {
  return parseJSX(rawJSX); // possibly throws an error
}

// ✅ good
const getMessageJSX = (rawJSX: string): JSX.Element => {
  try {
    return parseJSX(rawJSX);
  } catch (error) {
    logger.error(`Failed to parse JSX`);

    return <p>Unrecognized message</p>;
  }
}
```

## 2. CSS

#### 2.1. Use `rem-calc` function for "size" and "indent" values instead of hardcoding pixels

The `rem-calc` function from the [foundation framework](https://get.foundation/sites/docs/sass-functions.html#rem-calc) converts given value to `rem` units

>❓Why? To make UI more user-friendly for people who changed default font size in the browser settings. It looks better if fonts and indents are adapted as well

```scss
// ❌ bad
font-size: 32px;

// ✅ good
font-size: rem-calc(32);

// ❌ bad
margin-left: 8px;

// ✅ good
margin-left: rem-calc(8);
```

Make sure to use it only for `size` and `indent`. It's not necessary to specify, for example, `border-radius` in `rem`;

#### 2.2. Use `em` units for font-related properties

>❓Why? Otherwise it will be inconsistent if font size changes

```scss
// ❌ bad
font-size: rem-calc(24);
line-height: 32px;
letter-spacing: 2px;

// ✅ good
font-size: rem-calc(24);
line-height: 1.33em; // 32/24 = ~1.33
letter-spacing: 0.06em; // 2/32 = ~0.06
```

#### 2.3. Prefer internal atomic styles for indents

❌ bad
```jsx
<ul>
  <li className={styles.listItem}>
    // content
  </li>
</ul>
```

```scss
.listItem {
  margin-bottom: rem-calc(16);
}
```

❌ bad

Foundation provides set of atomic indents classes but they apply `!important` flag and are inconsistent with our system

```jsx
<ul>
  // applies margin-bottom: 1rem!important;
  <li className="margin-bottom-1">
    // content
  </li>
</ul>
```

✅ good
```jsx
<ul>
  // applies margin-bottom: 1rem;
  <li className="mb-16">
    // content
  </li>
</ul>
```

## 3. UI kit

The design system is built with [Storybook](https://storybook.js.org/) and deployed to [go/ui-kit](http://go/ui-kit) ([vpn](https://docs.google.com/document/d/1ybmBdRVKVhtMcbfIo54TxTCyDSat6Yp7IJdx4ekIO-U/edit#heading=h.bcp79hz4l863) is required)

#### 3.1 Follow process of adding new UI elements

In most cases it's possible to re-use existing code. Follow [this process](https://www.figma.com/file/ByAcTgdyQNDRct9HSyqh8R/Component-or-Style?type=whiteboard&node-id=0%3A1&t=jaOix2bCWscYlOMe-1) to make a decision

#### 3.2 DO NOT EXTEND visuals of existing components by adding props like `isLarge`, `isRounded`, `isSomethingElse`

In 99% of cases, we just don't need it. We should either keep the existing styles or change them in all places instead. Discuss it with responsible designers and decide how to proceed

#### 3.3 Always add new components to the UI kit. Follow figma naming

This just makes other developers' and designers' lives easier. Naming in Figma and code should be the same. It helps to identify existing components fast

#### 3.4 Document props using js-doc comments
Storybook parses js-doc and converts it into a description block

✅ good
```tsx
interface CustomProps {
  /** Describes the visual appearance of the button */
  mode?: ButtonMode;
  /** Describes the size of the button */
  size?: ButtonSize;
  /** Function Component Icon on the left side, typically used to render SVG icons. */
  LeftIcon?: FCIcon;
  /** Function Component Icon on the right side, typically used to render SVG icons. */
  ...
}
```

## 4. Raster assets images

#### 4.1 Follow guide when adding raster images to the website

To utilize benefits of images optimization provided by "image-handler" service we should store our images in the S3 bucket that is handled by this service. To do so - please follow the [guide](https://app.clickup.com/24383048/v/dc/q83j8-12520/q83j8-682675)

#### 4.2 Emojis naming

Since the most raster asset images on the website is emojis - it is important to keep them reusable and understandable (otherwise we'll face mess with such images). To achieve this goal - please follow these rules while adding images

##### 4.2.1 Follow DRY

Before adding any images to the website - please be sure that it is not exists here yet. Previously we had different folders with emoji images and it lead to situation with 2 (in some cases even 3) same images and it wasn't clear which of them should be used. To prevent such cases - please check `src/images/icons/emoji` folder to be sure that the image you want to add is not exist there. This rule is also applicable for other images (not only for the emojis), so please check all folders inside `src/images` to be sure that you won't create a duplicate.

##### 4.2.2 Naming rules

The best approach in icon naming - is to check how does this emoji named in Slack, we use Slack for communications and use a lot of emojis there, so it is relatively easy to us to recognise an emoji from the name taken from there.
Also for usage conveniency it is better to specify Emoji word in a name to make it clear that it is emoji. For example `IconWarning` and `IconEmojiWarning` are not the same (first one should be an icon with `svg` under the hood, second one - definitely raster one).

Example: 🎉

❌ Bad
```tsx
export const PartyIcon: FCImage = (props) => {
  /* Icon component code */
};
```
Explanation: this name is not following `IconEmoji...` pattern

❌ Bad
```tsx
export const MyFeatureSuccessModalIcon: FCImage = (props) => {
  /* Icon component code */
};
```
Explanation: the emoji icon is generic, so it's name shouldn't specify any relation to the feature or component

❗Please do not name generic icons according to it's usage. This approach lead to the mess in code and unnecessary duplicates


❌ Bad
```tsx
export const IconEmojiPartyHard: FCImage = (props) => {
  /* Icon component code */
};
```
Explanation: this name is not specific, and can be applied to any image

✅ good
```tsx
export const IconEmojiTada: FCImage = (props) => {
  /* Icon component code */
};
```
Explanation: this name follow `IconEmoji...` pattern and contain the name of this emoji in Slack
