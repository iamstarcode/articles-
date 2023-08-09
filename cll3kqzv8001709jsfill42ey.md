---
title: "Authentication in React Native with Expo Router 2 and Supabase"
datePublished: Wed Aug 09 2023 10:15:25 GMT+0000 (Coordinated Universal Time)
cuid: cll3kqzv8001709jsfill42ey
slug: authentication-in-react-native-with-expo-router-2-and-supabase
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1691575831443/b1b25ee3-a79d-42af-8833-6ebabaded430.png
tags: authentication, expo, supabase

---

# Introduction

React Native has emerged as a powerful and efficient framework for building cross-platform apps. One critical aspect of any app's functionality is user authentication, ensuring that the right users access the right resources securely. In this quest for a seamless authentication experience, developers often find themselves juggling multiple tools and libraries to strike the perfect balance between simplicity and robustness.

When developing React Native apps, efficient routing, navigation and seamless data management are essential aspects to consider. Over the years, [React Native Navigation](https://reactnavigation.org/) has helped with handling routing and navigation in React Native Projects, [Expo Router](https://docs.expo.dev/routing/introduction/) a file base router for React Native and web applications built upon [React Navigation suite](https://reactnavigation.org/) brings some of the routing concepts of the web to React Native applications. [Expo Router](https://docs.expo.dev/routing/introduction/) with [Supabase](https://supabase.com/) offers a powerful combination that simplifies both router and data handling, allowing developers to focus on building the core functionalities of their applications.

# Expo Router 2

In Expo's managed workflow, you can use the built-in navigation system provided by Expo's "react-navigation" library. Expo uses a customized version of "react-navigation" tailored to its specific requirements, and this is often referred to as "**Expo Router**." Expo Router handles navigation within the app, allowing users to move between different screens and manage the navigation stack.

It brings the best file-system routing concepts from the web to a universal application. If you have ever worked with React framework like NextJs 13 **app** folder, this will be easy to pick up as they are quite similar, allowing your routing to work across every platform. When a JSX file is added to the **app** directory, the file automatically becomes a route in your navigation stack.

# Expo Setup

The recommended way of setting up a new Expo project is using `create-expo-app`. It's fine to use the [Expo Go](https://docs.expo.dev/get-started/expo-go/) app to view our app on a mobile device, as this project doesn't require running native code.

```bash
npx create-expo-app@latest --template tabs@49
#or using pnpm
pnpm dlx create expo-app@latest --template tabs@49
```

Once setup has been completed, we need to install the Supabase JavaScript client library also, we can install this by running:

```bash
npx expo install @supabase/supabase-js
#or 
pnpm dlx expo install @supabase/supabase-js
```

To be able to run our project, we do that by running:

```bash
npx expo start
```

# **Supabase Setup**

In the course of this article, we will be using the Supabase local development setup. However, you can use the online dashboard to get up and running quickly. Developing locally comes with a few benefits such as faster development as there would be no need to connect with any network or **Ant** switch off your instance 😆.

Install [**Supabase CLI**](https://supabase.com/docs/guides/cli) based on your preferred platform and initialize the Supabase project in the root of the Expo project using the `supabase init` command, and run `supabase start`. Make sure you have Docker installed and running as this command when running the first time, downloads the required Docker images to run local containers of some Supabase services including PostgreSQL and a local dashboard. Once started and all services are running, you will see an output containing all local credentials which would look like this containing URLs and keys of your local projects.

```bash
Started supabase local development setup.

         API URL: http://localhost:54321
          DB URL: postgresql://postgres:postgres@localhost:54322/postgres
      Studio URL: http://localhost:54323
    Inbucket URL: http://localhost:54324
      JWT secret: super-secret-jwt-token-with-at-least-32-characters-long
        anon key: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
        service_role key: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

# Writing Code

Starting with Expo 49, [environment variables](https://docs.expo.dev/guides/environment-variables/) can be set using a `.env` file, in our environment variable we will be setting our Supabase API URL and anon key. Depending on how the Expo project is started, either with Web, Expo Go or development build, if it's running inside a Web browser, we will be able to access our Supabase API directly with the default URL `http://localhost:54321` when we set it in our `.env` file. However, if it's running on an Expo Go app or a development build, we have to make sure both the computer where the project is been built and the Expo Go app or the development build is on the same network and identify the IP address of the computer, usually starting with 192.168.\*\*\*.\*\*\*. To be able to access the local Supabase API from the device, we will need to substitute the localhost(`http://localhost:54321`) with the IP address of the computer on the network. To get the IP address of the computer, you can run this:

```bash
#Linux
ip addr show | grep 'inet '
#Windows or MacOs
ipconfig
```

We will need two environment variables for our Supabase Js Client `EXPO_PUBLIC_SUPABASE_URL` and `EXPO_PUBLIC_SUPABASE_ANON_KEY` .

To set environment variables in a .env file, follow these steps:

1. Create a new file named `.env` in the root directory of your project.
    
2. Open the .env file in a text editor.
    
3. Add the environment variables with their respective values in the .env file. Separate the variable names and values with the equals sign (=), without any spaces. For example:
    
    ```bash
    EXPO_PUBLIC_SUPABASE_URL=your-supabase-api-url
    EXPO_PUBLIC_SUPABASE_ANON_KEY=your-supabase-anon-key
    ```
    

We can set up a helper to initialize our Supabase client using the environment variables.

*./lib/supabate.ts*

```typescript
import 'react-native-url-polyfill/auto';
import { createClient } from '@supabase/supabase-js';
import { Database } from './database.types';
import * as SecureStore from 'expo-secure-store';

const ExpoSecureStoreAdapter = {
  getItem: (key: string) => {
    return SecureStore.getItemAsync(key);
  },
  setItem: (key: string, value: string) => {
    SecureStore.setItemAsync(key, value);
  },
  removeItem: (key: string) => {
    SecureStore.deleteItemAsync(key);
  },
};

export const supabaseClient = createClient<Database>(
  process.env.EXPO_PUBLIC_SUPABASE_URL ?? '',
  process.env.EXPO_PUBLIC_SUPABASE_ANON_KEY ?? '',
  {
    auth: {
      storage: ExpoSecureStoreAdapter as any,
      autoRefreshToken: true,
      persistSession: true,
      detectSessionInUrl: false,
    },
  }
);
```

We can now set up simple sign-in and sign-up forms.

/app/(auth)/sign-in.tsx

```typescript
import React from 'react';
import { useState } from 'react';
import { Alert } from 'react-native';
import { Box, Text, VStack, Icon } from 'native-base';
import Input from '@/components/ui/Input';
import Button from '@/components/ui/Button';
import { MaterialIcons } from '@expo/vector-icons';
import { supabaseClient } from '@/lib/supabase';

export default function SignIn() {
  const [view, setView] = useState<'sign-in' | 'sign-up'>('sign-in');
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [show, setShow] = useState(false);
  const [loading, setLoading] = useState(false);

  async function signInWithEmail() {
    setLoading(true);
    const { error } = await supabaseClient.auth.signInWithPassword({
      email: email,
      password: password,
    });

    if (error) Alert.alert(error.message);
    setLoading(false);
  }

  async function signUpWithEmail() {
    setLoading(true);
    const { error } = await supabaseClient.auth.signUp({
      email: email,
      password: password,
    });

    if (error) Alert.alert(error.message);
    setLoading(false);
  }
  return (
    <Box justifyContent='center' flex='1' px='3'>
      <Text fontSize={24} textAlign={'center'} fontWeight={'bold'} mt={5}>
        {view == 'sign-in' ? 'Sign in' : 'Sign up'}
      </Text>

      <VStack space={4} mt={5}>
        <Input
          autoComplete='email'
          placeholder='Email'
          type='text'
          onChangeText={(text: string) => setEmail(text)}
          value={email}
        />
        <Input
          onChangeText={(text: string) => setPassword(text)}
          value={password}
          type={show ? 'text' : 'password'}
          placeholder='Password'
          autoCapitalize={'none'}
          InputRightElement={
            <Icon
              as={
                <MaterialIcons name={show ? 'visibility' : 'visibility-off'} />
              }
              size={5}
              mr='2'
              color='muted.400'
              onPress={() => setShow(!show)}
            />
          }
        />
        {view == 'sign-in' ? (
          <Button
            isLoading={loading}
            size='lg'
            onPress={() => signInWithEmail()}
          >
            Sign in
          </Button>
        ) : (
          <Button
            isLoading={loading}
            size='lg'
            onPress={() => signUpWithEmail()}
          >
            Sign up
          </Button>
        )}
      </VStack>
      <Text
        mt='2'
        textAlign='right'
        color='blue.500'
        onPress={() => setView(view == 'sign-in' ? 'sign-up' : 'sign-in')}
      >
        {view == 'sign-in' ? 'Sign in' : 'Sign up'}
      </Text>
    </Box>
  );
}
```

In Expo router, any folder in the **app** folder automatically becomes a route, however, sometimes we could want to prevent a segment from showing in the URL by using the group syntax `()` .

When we created the Expo project with `create-expo-app` using the `--template tabs@49`, it setups a Tab with two screens, this tab will be our protected route before we get to this screen, we have to detect if a user is signed in or not before navigating to this screen, one of these ways this could be achieved is using a React context which will wrap around our root layout and handles routing based on the authenticated state.

*/context/AuthContext.tsx*

```typescript
import React from 'react';
import { useContext, useEffect, useState, createContext } from 'react';

import { AuthSession } from '@supabase/supabase-js';
import { useRouter, useSegments, useRootNavigationState } from 'expo-router';
import { supabaseClient } from '@/lib/supabase';

interface Props {
  children?: React.ReactNode;
}

export interface AuthContextType {
  session: AuthSession | null | undefined;
  authInitialized: boolean;
}

export const AuthContext = createContext<AuthContextType | undefined>(
  undefined
);

export function AuthProvider({ children }: Props) {
  const segments = useSegments();
  const router = useRouter();

  const [session, setSession] = useState<AuthSession | null>(null);
  const [authInitialized, setAuthInitialized] = useState(false);

  const navigationState = useRootNavigationState();

  useEffect(() => {
    if (!navigationState?.key || !authInitialized) return;
    const inAuthGroup = segments[0] === '(auth)';

    if (
      // If the user is not signed in and the initial segment is not anything in the auth group.
      !session?.user &&
      !inAuthGroup
    ) {
      router.replace('/(auth)/sign-in/');
    } else if (session?.user && inAuthGroup) {
      // Redirect away from the sign-in page.
      router.replace('/(tabs)/'); // to tabs
    }
  }, [session, segments, authInitialized, navigationState?.key]);

  useEffect(() => {
    if (authInitialized) return;

    supabaseClient.auth.getSession().then(({ data: { session } }) => {
      setSession(session);
    });

    const { data: authListner } = supabaseClient.auth.onAuthStateChange(
      async (_event, session) => {
        setSession(session);
        setAuthInitialized(true);

        if (_event == 'TOKEN_REFRESHED') {
          //Handle Accordinngly
        }
      }
    );

    return () => {
      authListner.subscription;
    };
  }, []);

  return (
    <AuthContext.Provider value={{ session, authInitialized }}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error(`useAuth must be used within a MyUserContextProvider.`);
  }
  return context;
};
```

In The `AuthProvider`, we will create a two `useEffect` hooks, one keeps track of the state of the Supabase `session`, route `segments` and the Expo router root navigation state `navigationState` . If authentication is not initialized and the root navigation state is not initialized, we return nothing, to prevent it from trying to use the router navigation without being initialized yet, and decides where to navigate based on which screen we are in if we have a user session or not.

The second `useEffect` is used to set an authentication listener using the `supabaseClient.auth.onAuthStateChange` callback to listen to authentication changes.

Once we have an auth context in place, we can then add it to our root `_layout.tsx` route in the app folder where shared elements persist between screens.

*/app/\_layout.tsx*

```typescript
import React from 'react';
import FontAwesome from '@expo/vector-icons/FontAwesome';
import {
  DarkTheme,
  DefaultTheme,
  ThemeProvider,
} from '@react-navigation/native';

import { useFonts } from 'expo-font';
import { Slot, SplashScreen } from 'expo-router';
import { useEffect } from 'react';
import { useColorScheme } from 'react-native';
import { AuthProvider, useAuth } from '../context/AuthContext';
import { SafeAreaProvider } from 'react-native-safe-area-context';

import { theme } from '../config/native-base-config';
import { Box, NativeBaseProvider } from 'native-base';

export {
  // Catch any errors thrown by the Layout component.
  ErrorBoundary,
} from 'expo-router';

export const unstable_settings = {
  // Ensure that reloading on `/modal` keeps a back button present.
  initialRouteName: '/(tabs)',
};

// Prevent the splash screen from auto-hiding before asset loading is complete.
SplashScreen.preventAutoHideAsync();

export default function RootLayout() {
  const [loaded, error] = useFonts({
    SpaceMono: require('../assets/fonts/SpaceMono-Regular.ttf'),
    ...FontAwesome.font,
  });

  // Expo Router uses Error Boundaries to catch errors in the navigation tree.
  useEffect(() => {
    if (error) throw error;
  }, [error]);

  useEffect(() => {
    if (loaded) {
      SplashScreen.hideAsync();
    }
  }, [loaded]);

  if (!loaded) {
    return null;
  }

  return (
    <AuthProvider>
      <NativeBaseProvider theme={theme}>
        <RootLayoutNav />
      </NativeBaseProvider>
    </AuthProvider>
  );
}

function RootLayoutNav() {
  const colorScheme = useColorScheme();
  const { session, authInitialized } = useAuth();

  if (!authInitialized && !session?.user) return null;

  return (
    <ThemeProvider value={colorScheme === 'dark' ? DarkTheme : DefaultTheme}>
      <SafeAreaProvider>
        <Slot />
      </SafeAreaProvider>
    </ThemeProvider>
  );
}
```

We can now start the app by running:

```bash
npx expo start
```

And then press the appropriate key for the environment we want to test the app in and you should see the completed app.

The complete source code can be found in this repository [here](https://github.com/iamstarcode/expo-router-2-with-supabase).