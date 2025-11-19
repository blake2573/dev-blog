## Containers all the way down

If you’re a web developer, you might be used to using [viewport units](https://web.dev/blog/viewport-units) to make your app more responsive. If you’ve been keeping up with CSS updates in the past few years, you might also have transitioned to using dynamic viewport units (dvh, dvw) instead, to help cater for address bar sizing on mobile browsers. However, when dealing with an app within an embedded environment like a Shopfiy app theme extension, you aren’t able to work with any of these, as you can’t rely on the viewport sizing to know the dimensions of where your specific app lives within the theme layout.

Instead, the best thing to do is make your entire app a container, and use [container query units](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Containment/Container_queries) everywhere that you would normally use viewport units.

For example, a common pattern for responsiveness might be to have a media query determine when to switch the layout up into a mobile view.

```tsx
@media (max-width: 800px) {
	...
}
```

Instead of this, however, you could quite easily update this into a container query instead

```tsx
@container (max-width: 800px) {
	...
}
```

Container queries are still evolving, so you might also be able to do orientation queries on your container (`@container (orientation: portrait)`) by the time you see this, or run style queries on your containers, the main point of this section is to remind you that media queries are not how you should trigger responsive layouts within a Shopify app - you should rely on container queries these days instead!

- Note: you can also container queries within your application for sub-components, to determine layout of their children based on their size, but this isn’t covered in this post as it’s more tangential of a thought. Definitely worth considering though!

## Tooltip positioning

Tooltip positioning, surprisingly, was quite frustrating to figure out how to get right. This, for me, was because it was my first experience dealing with an embedded application, and so I didn’t fully understand how the position of elements was impacted by parent containers. One thing I learned through this process, was that *fixed position isn’t always relative to the document* - [it can be thrown out by transformations on parent containers](https://dev.to/salilnaik/the-uncanny-relationship-between-position-fixed-and-transform-property-32f6). 

Being more familiar with standard web applications, I haven’t really had need to know what the specifics of fixed positioning is. Through the process of converting our app into a Shopify application, which can be embedded wherever a merchant decides, with whatever CSS the merchant decides on their store, I quickly discovered that whenever there is a transformation on an element, any child element of that parent can’t reliably use fixed positioning because the transform “resets” the position. This was honestly quite difficult to troubleshoot, but once I understood this it made so many other issues make sense, as you will learn below.

For tooltips, the best way to handle this was to manually look up the location of the target component, using the `.getBoundingClientRect()` function, and calculate the position of your tooltip based on this. Ideally you wouldn’t have to worry about this, and could just use [anchor positioning](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Anchor_positioning) and let the browser figure out where to place it, but until this becomes widely adopted within major browsers we are stuck doing it manually.

Some additional things to keep in mind while figuring out the tooltip positioning, is whether the tooltip overflows the page vertically or horizontally, as you should consider how to adapt to these issues so users can always see the entire tooltip, but in the future this will again hopefully be solved with CSS [position fallbacks](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/position-try-fallbacks).

### Modal positioning and transformations

Modal positioning is another piece of functionality has been another unforeseen complication when adapting our application into a Shopify environment. As explained above, for positioning tooltips, the modal positioning of `position: fixed;` doesn’t work as reliably within an embedded environment. In case you skipped over this, suffice it to say that using fixed positioning is impacted by *any* transformations on *any* parent component, so it can’t be relied upon for modal positions. 

This led me to another piece of functionality that I’ve not had reason to use before - portals. 

Now, I haven’t tried to implement these from scratch, so I don’t know the complexity for a general application, but React provides some built in functionality to make this much easier. (I assume some combination of `createElement` and `insertBefore` would provide a similar experience. I don’t doubt this becomes easier in future versions of Javascript!) The `createPortal` function helps to take a component and all its state, and render it in a specific place within the DOM. For my use-case, I was able to wrap my Modal component in a `createPortal` function, targeting the root of my application, so that instead of relying on `position: fixed;` I could use `position: absolute;`, and just position my modal based on where I want it to live within the app.

tl;dr: you could easily use your standard `inset: 50% 0 0 50%; transform: translate(-50%, -50%);`, and everything works like normal! This discovery was a life-saver to be honest, as before I found the concept of portals I was navigating fixed positions and event listeners to ensure everything stayed in place if you scrolled the page, which was a huuuuuge PITA. Trust me, just use portals and save yourself the trouble. If you aren’t using React it might be more of a pain, but still way easier than navigating the dynamic positions!

Modal transformations, on the other hand, can be used to add an additional smoothness to an application, while also accounting for a built-in cross-platform responsiveness that makes your site much easier to digest for users. What I like to do with modals, is build them so they have a smooth fade-in/out effect for desktop, but act like a slide-in panel on mobile - sliding in from the bottom of the screen. This helps provide a fairly common experience for users coming from other mobile applications, and helps make responsiveness easier.

To handle this slide-in functionality, I’ve previously employed a `translation` effect, starting the modal from `transform: translateY(100%);` where it starts off the bottom of the screen, and finishes at `transform: translateY(0);`. However, in a Shopify environment (or any embedded app environment) it doesn’t work to render components off-screen like this translation does.

Instead, I discovered a new piece of CSS that I haven’t had much chance to work with before now - the [`interpolate-size`](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/interpolate-size)  property. Adding `interpoliate-size: allow-keywords;` as a property on your modal component allows you to apply transitions on other properties, like `height` (eg: `transition: height 0.3s ease;`), more easily. Which is exactly what I was able to take advantage of to apply the same “slide-in” effect from within an embedded app environment in Shopify. It still *looks* like it’s sliding in from the bottom of the screen on mobile, but what’s actually happening is that the height of the modal is increasing until it reaches the appropriate height. And the best part is that this doesn’t *just* work within an embedded environment, this works in a standalone app as well, so has enhanced my default modal component’s animation to be more accommodating to more applications. I love to keep building out my own version of components that I can carry between applications as it makes a lot of boilerplate a lot easier to set up - perhaps one day I’ll set up my own personal component library repo. (Let me know if you’d be interested in this as I love to share!).

### Toasts

This is just a small section - as it is very much related to the fixed positioning points made above - except chances are that you are using a library dedicated to providing toasts into your application instead of building your own. 

For my example, I’m using [sonner](https://www.npmjs.com/package/sonner) for my toasts. It’s very simple, but on realising that `position: fixed;` wasn’t going to work, it was a bit of a pain to override the styles. Because I’d already been through this same issue with modals and tooltips however, I at least was aware of what the issue was, and was able to resolve it. Because most of these libraries have a context wrapper that wraps your whole application, you shouldn’t need to worry about portals, but should just be able to override the `position` property to be `position: absolute;` instead. This isn’t *strictly* true for all libraries, but hopefully gives an idea of how to conceptualise the required changes.

There are probably plenty more painful processes that I’ve come across in the transition from a regular website to a Shopify app, but these above have been the largest pain points - with fixed positioning causing the most pain. Alongside the above, I’ve also dealt with more fixed positioning pain with a user guide library (such as you would find with `reactour`, for example) which all seem to also use fixed positioning to find target components in the DOM, and as such be impacted by the same parent transform issues mentioned above.

Perhaps one day I will build my own library, or extend an existing one, that can better deal with portaling or positioning based on a root element parameter, instead of relying on fixed positioning and hoping it’s correct. But until then, I will continue to deal with the struggle of building things manually.

And that’s a wrap for this series of posts about converting a web app to deployable Shopify theme extension! I obviously haven’t covered the CI/CD process, or any of this, but I felt this was well enough documented that it didn’t warrant extra comment. If anybody wants a sample repository, or more information, feel free to reach out to me directly. I hope you’ve enjoyed following along on this journey, and that perhaps it’s helped you build your own Shopify app now or in the future.


### Code snippets

The below code snippets are customised for this blog, but represent essentially the same logic flow as is required, and should be able to be incorporated to your site easily enough.

1. A modal component that is compatible with Shopify

```tsx
// Modal.tsx
import styles from './Modal.module.scss'

// Note: this duration must match the $animation-duration variable in Modal.module.scss
export const modalAnimationDurationMs = 450

type ModalProps = {
  visible: boolean
  onClose: () => void
  closeButton?: boolean
  disableClose?: boolean
  className?: string
  children: ComponentChildren
}

export const Modal = ({
  children,
  className,
  visible,
  onClose,
  closeButton,
  disableClose,
}: ModalProps) => {
  // A ref to whatever your root component is
  const { applicationRef } = useApplicationContext()

  const modalRef = useRef<HTMLDivElement>(null)
  const modalContentRef =
    useRef<HTMLDivElement>(null)
  const [isOpen, setIsOpen] = useState(visible)
  const [isClosing, setIsClosing] =
    useState(false)

  // A lot of this is to handle closing transitions
  // If CSS has improved enough by the time you read this, you may not need everything here
  useEffect(() => {
    if (visible) {
      setIsOpen(true)
      if (modalRef.current)
        modalRef.current.scrollTop = 0 // Scroll to top on open
      // Mark that a modal is open to prevent tooltips from hiding
      if (applicationRef.current) {
        applicationRef.current.dataset.modalOpen =
          'true'
      }
    } else {
      setIsClosing(true)
      setTimeout(() => {
        setIsClosing(false)
        setIsOpen(false)
        // Remove modal open marker after closing animation
        if (applicationRef.current) {
          delete applicationRef.current.dataset
            .modalOpen
        }
      }, modalAnimationDurationMs)
    }
  }, [visible, applicationRef])

  return (
    isOpen &&
    applicationRef.current &&
    createPortal(
      <Fragment>
        <div
          ref={modalRef}
          className={cl(styles.modal, className, {
            [styles.isClosing]: isClosing,
            [styles.isOpen]: isOpen,
          })}
        >
          {closeButton && (
            <button
              className={styles.closeButton}
              type='button'
              onClick={onClose}
              title='Close modal'
            >
              <Icon name='close' size={20} />
            </button>
          )}
          <div
            ref={modalContentRef}
            className={styles.content}
          >
            {children}
          </div>
        </div>
        <div
          className={cl(styles.backdrop, {
            [styles.isClosing]: isClosing,
            [styles.isOpen]: isOpen,
          })}
          onClick={
            disableClose ? undefined : onClose
          }
        />
      </Fragment>,
      applicationRef.current,
    )
  )
}
```

```scss
// Modal.module.scss
// Note: This duration must match the modalAnimationDurationMs variable in Modal.tsx
$animation-duration: 450ms;

.modal {
  background-color: $app-color-background-2;
  border: none;
  border-radius: 26px;
  cursor: auto;
  height: auto;
  inset: 50% auto auto 50%;
  interpolate-size: allow-keywords;
  margin: auto;
  max-height: 80cqh;
  max-width: 80cqw;
  outline: none;
  overflow: hidden;
  padding: 0;
  position: absolute;
  transform: translate(-50%, -50%);
  z-index: 12;

  &.isClosing {
    animation: elementFadeOut $animation-duration
      ease forwards;
  }

  &::backdrop {
    animation: backgroundFadeOut
      $animation-duration ease forwards;
    background: rgba(0, 0, 0, 0.5);
  }

  &.isOpen:not(.isClosing) {
    @include opacity(1);

    &::backdrop {
      animation: backgroundFadeIn
        $animation-duration ease forwards;
    }
  }

  .closeButton {
    background-color: transparent;
    color: $app-color-mono-text;
    height: 2em;
    padding: 0;
    position: absolute;
    right: 0.5em;
    top: 0.5em;
    width: 2em;

    &:hover {
      opacity: 0.4;
    }
  }

  @include onMobile {
    border-bottom-left-radius: 0;
    border-bottom-right-radius: 0;
    inset: auto auto 0 50%;
    margin: unset;
    max-height: calc(75cqh);
    max-width: 100cqw;
    @include opacity(1);
    transform: unset;
    transition: height 0.3s ease;
    width: 100%;

    &.isClosing {
      animation: slideOutToBottom
        $animation-duration ease forwards;
    }

    &.isOpen:not(.isClosing) {
      animation: slideInFromBottom
        $animation-duration ease forwards;
    }
  }

  .content {
    max-height: calc(80cqh - 3em);
    overflow: auto;
    padding: 1.5em;
    width: 100%;

    @include onMobile {
      max-height: calc(90cqh - 3em);
    }
  }
}

.backdrop {
  animation: backgroundFadeOut $animation-duration
    ease forwards;
  background: rgba(0, 0, 0, 0.5);
  inset: 0;
  pointer-events: none;
  position: absolute;
  z-index: 11;

  &.isOpen:not(.isClosing) {
    animation: backgroundFadeIn
      $animation-duration ease forwards;
    pointer-events: all;
  }
}

@keyframes slideInFromBottom {
  from {
    height: 0;
    transform: translateX(-50%) translateY(0);
  }
  to {
    height: auto;
    transform: translateX(-50%) translateY(0);
  }
}

@keyframes slideOutToBottom {
  from {
    height: auto;
    transform: translateX(-50%) translateY(0);
  }
  to {
    height: 0;
    transform: translateX(-50%) translateY(0);
  }
}

@keyframes backgroundFadeIn {
  from {
    background: rgba(0, 0, 0, 0);
  }
  to {
    background: rgba(0, 0, 0, 0.5);
  }
}

@keyframes backgroundFadeOut {
  from {
    background: rgba(0, 0, 0, 0.5);
  }
  to {
    background: rgba(0, 0, 0, 0);
  }
}

@keyframes elementFadeOut {
  from {
    @include colors.opacity(1);
  }
  to {
    @include colors.opacity(0);
  }
}
```