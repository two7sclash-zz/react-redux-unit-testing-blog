# Comprehensive React/Redux Unit Testing
Here is a collection of the best practices I've identified for circumspect and pithy React/Redux Unit Testing.

This article assumes you are already happy and comfortable using React, Redux, and know the basic testing methods from Enzyme.

## Actions & Async Action Creators
All we care about here is that the correct action creator was called and it returned the right action. So unit tests should only know about actions/events and state. 

For async action creators using Redux Thunk (or other middleware), mock the (minimally) required Redux store for testing. If needed, you can apply the middleware to said store using `redux-mock-store`. You can also use fetch-mock to mock the HTTP requests, but that might be overkill. I've found it preferable to simply use a `mockServiceCreator` function with a suite of body fixtures.

  Here's a somewhat complicated async action creator we would like to test, that does one or two factor authentication:
  ````
  import { login as loginService } from '../services/login';
  // there are several other async action creators I've left out, as well as library defs and constant imports
  ...
  export const login = ({ username, password }, service = loginService) => (dispatch) => {
  // note default service value
    dispatch({ type: ACTION_LOGIN_REQUEST });
    return service({ username, password })
      .then(({ status, phoneNumbers }) => {
        if (status === ONE_FA) {
          dispatch({
            type: ACTION_STORE_PHONE_NUMBERS,
            payload: { phoneNumbers },
          });
          dispatch({ type: ACTION_SET_LOADING_FALSE });
          dispatch(push('/select-otp-delivery'));
        } else if (status === TWO_FA) {
          dispatch({ type: ACTION_SET_USER_LOGGED_IN });
        }
      })
      .catch(() => dispatch({ type: ACTION_LOGIN_FAILURE }));
  };
  ````
  The entire reducer might look something like:
  ```
  const getInitialState = () => ({
    phoneNumbers: [],
    isLoggedIn: false,
    loginError: null,
    isLoading: false,
  });
  
  export default (state = getInitialState(), { payload, type }) => {
    switch (type) {
      case ACTION_LOGIN_REQUEST:
        return { ...state, isLoading: true };
      case ACTION_LOGIN_FAILURE:
        return {
          ...state,
          loginError: LOGIN_FAILURE_MESSAGE,
          isLoading: false,
        };
      case ACTION_STORE_PHONE_NUMBERS:
        return { ...state, phoneNumbers: payload.phoneNumbers };
      case ACTION_SET_USER_LOGGED_IN:
        return { ...state, isLoggedIn: true, isLoading: false };
      case ACTION_INITIATE_OTP_REQUEST:
        return { ...state, isLoading: true };
      case ACTION_VALIDATE_OTP_REQUEST:
        return { ...state, isLoading: true };
      case ACTION_SET_LOADING_FALSE:
        return { ...state, isLoading: false };
      default:
        return state;
    }
  };
  ```
  And here is the base of the testing suite for the entire authentication flow, using redux-mock-store and thunks:
  
  ```
  import configureMockStore from 'redux-mock-store';
  import thunk from 'redux-thunk';
  
  const middlewares = [thunk];
  const mockStore = configureMockStore(middlewares);
  
  // allows us to easily return reponses and/or success/fail for a thunk that calls a service
  const mockServiceCreator = (body, succeeds = true) => () =>
    new Promise((resolve, reject) => {
      setTimeout(() => (succeeds ? resolve(body) : reject(body)), 10);
    });
    
    describe('authenticate action', () => {
      let store;
      // set up a fake store for all our tests
      beforeEach(() => {
        store = mockStore({ phoneNumbers: [] });
      });
    
   ```
  The above action creator is tested thus:
  ```
    describe('when a user logs in', () => {
      it('fires a login request action', () =>
        store
          .dispatch(login(
            { username: 'user', password: 'pass' },
            mockServiceCreator(LOGIN_SUCCESS_OTP_REQUIRED_BODY),
          ))
          .then(() => expect(store.getActions()).toContainEqual({ type: ACTION_LOGIN_REQUEST })));
  
      describe('when login succeeds and OTP is required', () => {
        beforeEach(() =>
          store.dispatch(login(
            { username: 'user', password: 'pass' },
            mockServiceCreator(LOGIN_SUCCESS_OTP_REQUIRED_BODY),
          )));
  
        it('dispatches action to store phone numbers and updates the route', () => {
          const actions = store.getActions();
          const { phoneNumbers } = LOGIN_SUCCESS_OTP_REQUIRED_BODY;
          expect(actions).toContainEqual({
            type: ACTION_STORE_PHONE_NUMBERS,
            payload: { phoneNumbers },
          });
          expect(actions).toContainEqual(push('/select-otp-delivery'));
        });
      });
     ... 
     MORE TESTS
```

## Reducers
A reducer should just return a new state object after applying an action to the previous state. 

So for the reducer I introduced above, we can simply do: 
````
const initialState = {
  phoneNumbers: [],
  isLoggedIn: false,
  loginError: null,
  isLoading: false,
};
const loggedInState = {
  phoneNumbers: [],
  isLoggedIn: true,
  loginError: null,
  isLoading: false,
};

describe('authenticate reducer', () => {
  it('returns the initial state', () => {
    expect(reducer(undefined, {})).toEqual(initialState);
  });

  it('handles login request', () => {
    expect(reducer(initialState, { type: ACTION_LOGIN_REQUEST })).toEqual({
      ...initialState,
      isLoading: true,
    });
  });

  it('handles login failure', () => {
    expect(reducer(initialState, { type: ACTION_LOGIN_FAILURE })).toEqual({
      ...initialState,
      loginError:
        'Sorry, it looks like the Username and/or Password you provided does not match our records',
    });
  });

  it('handles successful 1fa login', () => {
    expect(reducer(initialState, {
      type: ACTION_STORE_PHONE_NUMBERS,
      payload: { phoneNumbers: ['1234567890'] },
    })).toEqual({
      ...initialState,
      phoneNumbers: ['1234567890'],
    });
  });
  ...
  MORE TESTS
````

## Components
* Shallow render whenever possible! 
    * Constrain yourself to testing a component as a unit; `shallow`  ensures that your tests aren't indirectly asserting on behavior of child components. (Here's a great refresher as to the difference between `shallow`, `mount` and `render`, and the lifecycle methods involved: https://gist.github.com/fokusferit/e4558d384e4e9cab95d04e5f35d4f913) 
    * Remember that event propagation is unavailable here. 
    * It can helpful to use an enhanced shallow testing function, which allows you to shallow render until a specific selector is reached.  The gold standard is [Material UI's implementation](https://material-ui-next.com/guides/testing/#api). Such an inhancement improves shallow rendering for use with higher-order component. Discussions of other implementations of such a "mountUntil" type utility can be found here: https://github.com/airbnb/enzyme/issues/250
    * If `componentDidMount` or `componentDidUpdate` needs testing, or you have child components that directly interact with APIs, use `mount`. `Mount` by design,  should simulate real DOM behavior if needed, but beware that there are [limitations to what JSDOM actually provides](https://github.com/jsdom/jsdom#caveats).

* Test to make sure all sub-components of the component you are testing at least render (using manually passed, logic-dictating props as needed).

* Test the component when not connected to Redux. Directly export  your unconnected component alongside default export of the connected component. E.g.: 
    ```
    // raw, unconnected component for testing
    export function HeaderLinks(props) {
    ...
        return (
        <Grid container item md={10} sm={8} xs={6} className={classes.navi}>
              <HeaderMenu renderMenuLinks={() => menuLinks} />
            </Grid>
            )
    }
    // connected (or any other sort of HOC component, etc) for use in App
    export default connect(null, mapDispatchToProps)(compose(withStyles(styles), withWidth())(HeaderLinks));
    ```
    * It is not necessary to test that our mapDispatchToProps is properly passing a function (login, submit, click etc.) to the connected component, because Redux is already responsible for this.
    * If you want to test its interaction with Redux, you can wrap it in a <Provider> with a store created specifically for this unit test. (But this seems more like Integration/Journey testing.)

* When we simulate a DOM event, we need to pass the preventDefault function in our event object or we will get an error. 
    * This is because the  function will call event.preventDefault(), and if we don’t include it an error occurs. Once we simulate the submit event, we can then test our event handler with a mock function to see if it was called (purpose here isn't to test React event wiring, or browser event handling). E.g.:
    ```
    describe('when the application is submitted', () => {
          beforeEach(() => {
            component.find('form').simulate('submit', { preventDefault() {} });
          });
     ```
 * There's no need to test that React will trigger a "change" event when a element is clicked (i.e. a checkbox) - just shallow-render your Component, and assert that appropriate attribute (i.e. `onChange`) matches the prop or state you pass in manually to your test. You should be using `state` to store the current state of a given element element and base your rendering on that. [Remember how Eventing works in a proper React setup](https://www.kirupa.com/react/events_in_react.htm).


## Connected Components

* While testing the redux wrapped (connected) HOC is not recommended, the nature of our app mean you will often to search for the existence of components wrapped with `connect`, `withStyles` etc.
* But what about `mapStateToProps?` This can be exported and tested in isolation as a pure function if desired. My response to this method is a resounding "Meh." Really, if you Shallow render a container and test that the appropriate actions are dispatched, you're already testing this. 

## Forms

Per above, we really should not be changing state/props through Eventing. Rather we can test that each form element event handler fires the appropriate prop function, and that it perhaps changes 'state' in the appropriate way. Then we can simulate the entire form getting submitted. Again, _Enzyme's shallow( ... ).simulate() does not bubble events; it gets the event handler and invokes it. 

E.g.:

```component.find('form').simulate('submit', { preventDefault: jest.fn() });```

Of course, forms are only useful when filled, and there can be a myriad of different error handling methods and routings in result. The naive testing strategy would be to use `mount` several time to manipulate form fields to set a given state to test the results of submit against. Or maybe to `simulate` a change event of several fields. Instead, we should set these internals directly as we have already tested the event handlers of individual form elements. `setState` is available, but per the [docs](http://airbnb.io/enzyme/docs/api/ShallowWrapper/setState.html)  "This method is useful for testing your component in hard to achieve states, however should be used sparingly. If possible, you should utilize your component's external API (which is accessible via .instance()) in order to get it into whatever state you want to test, in order to be as accurate of a test as possible. This is not always practical, however."

So _instead_ of something like:

```
beforeEach(() => {
    onAddExternalAccount = jest.fn();

    component = shallow(<AddExternalAccount onAddExternalAccount={onAddExternalAccount} classes={styles} />);
 
    component.find('#routing-number').simulate('change', { target: { value: '1234567890' } });
    component.find('#account-number').simulate('change', { target: { value: '0987654321' } });
    component.find('#account-type').simulate('change', { target: { value: 'Checking' } });
  });
```
_Rather_ do:
```
beforeEach(() => {
    onAddExternalAccount = jest.fn();

    component = shallow(<AddExternalAccount onAddExternalAccount={onAddExternalAccount} classes={styles} />);

    component.instance().setAccountNumber({ target: { value: '1234567890' } })
    // etc. for all the fields
    ...
  });
```

If it isn't clear, above we had something like the below as a method on the `AddExternalAccount` component:
```
 setRoutingNumber = event => {
    const input = event;
    const { externalAccountInfo } = this.state;
    const { value: routingNumber } = input.target;
    this.setState({
      externalAccountInfo: { ...externalAccountInfo, routingNumber },
    });
  };
```

And the component looked something like:

```
<TextField
                    fullWidth
                    id="routing-number"
                    type="text"
                    label="Bank Routing Number"
                    placeholder="9 digit number"
                    onChange={this.setRoutingNumber}
                  />
 ```



In other words, we should try to grab our form, and use the real state setting methods called by the event handlers to drive our form's data for a given test.

Lastly, Enzyme makes it really easy to test a component like this but it was non-obvious how to test the default value of the form. 
```
it('defaults to 5 minimum payments', function () {
  const wrapper = shallow(
    <PaymentsTab />
  )
  assert.equal(wrapper.find('[id="visitSelector"]').node.value, 5)
})
```

Here we can just shallow render the component, call find() on the test id and then use .node.value to get the default value.

## Summary
???

 
 
