# Snapcart Developer Application Project Report

This is a comprehensive report on the implemented codebase for the Snapcart project based on files and folders pulled from the repository, in relation to the product design found on Figma. Contained in this report is a list of things that might have been done wrongly and/or could be done better, and how I intend to fix these things if given the chance. The format is the given observation in one paragraph and my proposed solution/fix in the following paragraph.


## App Name & Launch Icons
After pulling from the project repository, installing all necessary dependencies and running the project, my first observation was spotting a wrong app name (contrary to what is in the design) and the default launch icon for the application.

To update the app name, I would navigate to the `strings.xml` file responsible for displayed app name on Android. That is: `/android/app/src/main/res/values/strings.xml`, and in there change the app name from  `instacom` to the intended `Snapcart`. For the case of iOS, I would update the application's `CFBundleDisplayName` from `instacom` to `Snapcart` by navigating to `/ios/instacom/Info.plist`. To create launch icons, I would either request for a launch icon image asset (.png format preferably and with a 300 x 300 dimension) from the designer or create one from the designs. With that available, I would install a dependency: [Bam Tech React Native Make](https://www.npmjs.com/package/@bam.tech/react-native-make). A command on this library helps in generating and setting launch icons for both platforms. Closing and rebuilding the project would effect these changes. 


## Splash Screen
On opening the application for the first time, my next observation was a missing splash screen for the application. Apart from giving users a better experience, Splash screens help reduce user anxiety while waiting for the application to load.

To resolve this, I would save the actual splash screen asset from the design (.png format preferably) in the assets folder and delete the existing one. Then proceed to install another dependency: [React Native Splash Screen](https://www.npmjs.com/package/react-native-splash-screen). While the initially installed [Bam Tech React Native Make](https://www.npmjs.com/package/@bam.tech/react-native-make) also helps in generating and setting a splash screen for both platforms, the [React Native Splash Screen](#) is what helps on the native side to show this to the users and hide it once initializing actions over the native bridge are completed.


## Auth Screen(s)
A few observations on the auth screens include contrast in implementation for screens layout, form inputs and buttons with respect to the design, missing form validation, no means to notify users of wrong input or response of network requests

I would create a shared component for the layout taking note of all possible passed props then wrap the respective `Login`, `Forgot Password` and `Register` screens with this. For the text input and button, a separate component will be created for each replicating the exact pixel-perfect look in the design and housing the required logic. To handle form validation, the existing hooks (useRegisterErrors & useLoginErrors) could either be optimized or a new useForm hook created to enforce input format by notifying users of the expected format. A perfect way to do this is leveraging the options offered by the [React Native Flash Message](https://www.npmjs.com/package/react-native-flash-message) package, in effect, improve user experience. A typical `useForm` hook will look like the image below: 

![Form Validation Hook]('./useForm-hook.png' "useForm")


## Exposed URL Parameter
While initiating the Apollo Client, the base URL appeared to be passed to the link property in the raw format, that is: `https://*******.herokuapp.com/graphql`.

This works fine but might be risky in the future should there be a case of security breach on the product. To resolve this, I would create an environment variable (.env) file at the root level of the project. Environment variables provide a good way to set application execution parameters such as URIs and API keys that are used by different methods and parts of the application. I would install a dependency: [React Native Dotenv](https://www.npmjs.com/package/react-native-dotenv) and after configuring the `babel.config.js` file accordingly, variables can be accessed in any part of the application.


## Apollo Client Setup
Additionally, the `Providers.js` file referenced in `App.js` is where the Apollo Client is initialized and passed to the ApolloProvider wrapping the entire application. While this would work, the Apollo Client setup is something I would have done differently because there is a more efficient way to do this than the current approach. Moreso, with the current implementation, we're having to trigger a `setToken` function from the `Login` and `Register` screens (after a successful login or registration respectively) which are deeply nested in the navigation routing heirarchy and this results in needless prop drilling. Also, there was a wrong destructuring in the same file when trying to get the token from the AsyncStorage. Below is what we have:

```
  const fetchSession = async () => {
    const session = await AsyncStorage.getItem('@instacom-graphql:session');
    if (session) {
      const sessionObj = JSON.parse(session);
      const { token } = sessionObj;
      setTokenString(token);
    }

    setLoading(false);
  };
```

The destructured `token` here would always be undefined due to the way token is set to the AsyncStorage in both `Login.js` and `Register.js`. Rather, the block should be updated to:

```
  const fetchSession = async () => {
    const session = await AsyncStorage.getItem('@instacom-graphql:session');
    if (session) {
      const sessionObj = JSON.parse(session);
      setTokenString(sessionObj);
    }

    setLoading(false);
  };
```

To give us a better approach, I would rewrite the `makeApolloClient` function exported to cater for attaching a user's authentication token to the GraphQL execution context. Our updated `apollo.js` will look like below:

![Apollo Client Setup]('./makeApolloClient.png' "MakeApolloClient in apollo.js")

With this, not only can the `fetchSession` code block above and the accompanying useEffect function be moved to the Routes file, our Providers file becomes much leaner with only the necessary code as can be seen in the block below:

```
const Providers = () => {
  const client = makeApolloClient();

  return (
    <ApolloProvider client={client}>
      <Routes />
    </ApolloProvider>
  );
};
```

In addition, we do not need to pass up `setToken` from Login > AuthStack > Routes anymore. Consequently, to track the loggedIn state in the Routes file, we can then leverage on the use of Reactive Variables provided out of the box by Apollo Client. It can be initialized as false and then set to true when: 1). a user logs in, 2). a user signs up, or 3). a token exists in the AsyncStorage already


## Other Notable Observations
Lastly, a few other things worthy of note include but not limited to the following:

### Performance
In the `Register.js` file, I found a case to where optimizing the useMutation call would improve performance. We can achieve this by wrapping the mutation call in a try-catch block and managing the loading state locally. Below is an example of what we have currently:

```
  const [createAccount, { data, loading }] = useMutation(CREATE_ACCOUNT);
  const signup = () => {
    const { password, email, name } = formState;
    createAccount({ variables: { email, password, name } });
  };

  if (data) {
    AsyncStorage.setItem(
      '@instacom-graphql:session',
      JSON.stringify(data.signup.token)
    );
    setToken(data.signup.token);
  }
```

And below is what we would have after the update

```
  const [createAccount] = useMutation(CREATE_ACCOUNT);
  const signup = async () => {
    const { password, email, name } = formState;
    setLoading(true);
    try {
      const { data } =  await createAccount({ variables: { email, password, name } });
      if (data) {
        AsyncStorage.setItem(
          '@instacom-graphql:session',
          JSON.stringify(data.signup.token)
        );
        isLoggedInVar(data.signup.token)       // Reactive Variable
        setLoading(false);
      }
    } catch (error) {
      setLoading(false);
    }
  };
```

### Architecture
In the `AuthStack.js` file, I noticed that the rendered component for the `headerBackImage` property was reused multiple times and in multiple files.

A cleaner solution for this in the future is to move this into a separate file housing the component. Which can then be referenced from anywhere.

### Custom Icons
Oftentimes, it is not uncommon for some or most of the icons in the UI design for a project to be custom icons and not among those that come bundled with react-native-vector-icons. This is usually among the first set of things I like to sort in each project before it gets bigger. I take my time in going through the designs, exporting all custom icons in .svg format. Using Icomoon, an app for making custom icon fonts or SVG sprites, I would then generate a font from these custom icons and save the generated lightweight JSON file in the project directory. With this, we can do away with the actual svgs and use our icons anywhere in the application same way we would use a vector icon.
