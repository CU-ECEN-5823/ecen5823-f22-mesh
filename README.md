# ecen5823-f22-mesh

Author : Dave Sluiter
Date   : 3/14/2022

Starter code for the Bluetooth Mesh Assignment based on Gecko SDK 3.2.3

There are 2 Simplicity Studio projects in this repository:

ecen5823-f22-mesh-cp/  - Client-Publisher, this originated as the SiLabs 
                         soc_btbesh_switch example. The switch example publishes messages  
                         to a Server-Subscriber. 
                         
ecen5823-f22-mesh-ss/  - Server-Subscriber, this originated as the SiLabs 
                         soc_btbesh_light example. The example subscribes to messages 
                         from a Client-Server. 


************************************************
Document soc_btmesh_switch code
************************************************

This file documents a portion of the the construction of the SiLabs example 
project: soc_btmesh_switch which was the basis of this starter code for the
mesh assignment. Down below are summaries of the changes made to both the soc_btmesh_switch
and the sc_btmesh_light example code. 

soc_btmesh_switch became ecen5823-f22-mesh-cp   (client-publisher)
soc_btmesh_light  became ecen5823-f22-mesh-cp   (server-subscriber)



For soc_btmesh_switch, the relevant installed Software Components are:

Utility / Button Press
  API for handling button presses of various lengths
  Has Configure controls for setting the duration of Short, Medium and Long button presses
  File = app_button_press_config.h 

Driver / Button / Simple Button
  btn0
  btn1 
  Have Configure controls for setting Mode of operation, set to: Interrupt
  for both, and also assigning the the port and pin numbers to use.
  PF6 = button 0
  PF7 = button 1
  File = sl_simple_button_btn0_config.h
  File = sl_simple_button_btn1_config.h
  
Driver / Button / Generic Button API
This component provides a generic button API. In addition, a button driver  
implementation component such as the Simple Button component should be 
included in the project to implement full button handling.

Also has the API documentation, + links to webpage documentation. Not replicating all
of that here. Just a few summary notes:

// ### --------------------------------------------------------------

There is currently one type of button supported by the button driver: Simple Button Driver

Both the interrupt and polling methods obtain the button state for the user by calling 
sl_button_get_state(). 

// ### --------------------------------------------------------------


Ok, now trace functionality "upwards", starting with 

GPIO_EVEN_IRQHandler()
GPIO_ODD_IRQHandler()

Application definitions of these functions are defined in : gpiointerrupt.c 


/* Array of user callbacks. One for each pin interrupt number. */
static GPIOINT_IrqCallbackPtr_t gpioCallbacks[32] = { 0 };


// For reference 
// *************************************************************************
void GPIOINT_Init(void)
{
  if (CORE_NvicIRQDisabled(GPIO_ODD_IRQn)) {
    NVIC_ClearPendingIRQ(GPIO_ODD_IRQn);
    NVIC_EnableIRQ(GPIO_ODD_IRQn);
  }
  if (CORE_NvicIRQDisabled(GPIO_EVEN_IRQn)) {
    NVIC_ClearPendingIRQ(GPIO_EVEN_IRQn);
    NVIC_EnableIRQ(GPIO_EVEN_IRQn);
  }
}

// Even 
// *************************************************************************
void GPIO_EVEN_IRQHandler(void)
{
  uint32_t iflags;

  /* Get all even interrupts. */
  iflags = GPIO_IntGetEnabled() & _GPIOINT_IF_EVEN_MASK;

  /* Clean only even interrupts. */
  GPIO_IntClear(iflags);

  GPIOINT_IRQDispatcher(iflags);
}

// Odd
// *************************************************************************
void GPIO_ODD_IRQHandler(void)
{
  uint32_t iflags;

  /* Get all odd interrupts. */
  iflags = GPIO_IntGetEnabled() & _GPIOINT_IF_ODD_MASK;

  /* Clean only odd interrupts. */
  GPIO_IntClear(iflags);

  GPIOINT_IRQDispatcher(iflags);
}


// *************************************************************************
static void GPIOINT_IRQDispatcher(uint32_t iflags)
{
  uint32_t                  irqIdx;
  GPIOINT_IrqCallbackPtr_t  callback;
  // defined in gpiointerrupts.h:
  //    typedef void (*GPIOINT_IrqCallbackPtr_t)(uint8_t intNo);

  /* check for all flags set in IF register */
  while (iflags != 0U) {
    irqIdx = SL_CTZ(iflags); // turn flag bits into an index

    /* clear flag*/
    iflags &= ~(1UL << irqIdx);

    callback = gpioCallbacks[irqIdx]; // get the address of the callback
    if (callback) {
      /* call user callback */
      callback((uint8_t)irqIdx);
    }
  }
}


So this is the mechanism by which the application ISRs are handled. Now figure out how
the addresses of the ISR routines are loaded into gpioCallbacks[32]. This is handled by:

// *************************************************************************
void GPIOINT_CallbackRegister(uint8_t intNo, GPIOINT_IrqCallbackPtr_t callbackPtr)
{
  CORE_ATOMIC_SECTION(
    /* Dispatcher is used */
    gpioCallbacks[intNo] = callbackPtr;
    )
}


GPIOINT_CallbackRegister() is called from : sl_simple_button.c

// *************************************************************************
sl_status_t sl_simple_button_init(void *context)
{
  sl_simple_button_context_t *button = context;

  CMU_ClockEnable(cmuClock_GPIO, true);

  GPIO_PinModeSet(button->port,
                  button->pin,
                  SL_SIMPLE_BUTTON_GPIO_MODE,
                  SL_SIMPLE_BUTTON_GPIO_DOUT);

  button->state = ((bool)GPIO_PinInGet(button->port, button->pin) == SL_SIMPLE_BUTTON_POLARITY);

  if (button->mode == SL_SIMPLE_BUTTON_MODE_INTERRUPT) {
    GPIOINT_Init();
    GPIOINT_CallbackRegister(button->pin,
                             // Callback function that handles the interrupts 
                             (GPIOINT_IrqCallbackPtr_t)sli_simple_button_on_change); 
    GPIO_ExtIntConfig(button->port,
                      button->pin,
                      button->pin,
                      true,  // rising edge
                      true,  // falling edge
                      true); // enable IRQs
  }

  return SL_STATUS_OK;
}




We have two paths to follow:
  a) The initialization code  : sl_simple_button_init()
  b) The callback IRQ handler : sli_simple_button_on_change()
  



************************************************
Follow the initialization code
************************************************

sl_simple_button_init() is referenced from : autogen/sl_simple_button_instances.c

defining/filling out these data structures:

// Button 0
sl_simple_button_context_t simple_btn0_context = {
  .state = 0,
  .history = 0,
  .port = SL_SIMPLE_BUTTON_BTN0_PORT,
  .pin = SL_SIMPLE_BUTTON_BTN0_PIN,
  .mode = SL_SIMPLE_BUTTON_BTN0_MODE,
};

const sl_button_t sl_button_btn0 = {
  .context = &simple_btn0_context,
  .init = sl_simple_button_init,            // << callback 
  .get_state = sl_simple_button_get_state,
  .poll = sl_simple_button_poll_step,
  .enable = sl_simple_button_enable,
  .disable = sl_simple_button_disable,
};


// Button 1
sl_simple_button_context_t simple_btn1_context = {
  .state = 0,
  .history = 0,
  .port = SL_SIMPLE_BUTTON_BTN1_PORT,
  .pin = SL_SIMPLE_BUTTON_BTN1_PIN,
  .mode = SL_SIMPLE_BUTTON_BTN1_MODE,
};

const sl_button_t sl_button_btn1 = {
  .context = &simple_btn1_context,
  .init = sl_simple_button_init,            // << callback
  .get_state = sl_simple_button_get_state,
  .poll = sl_simple_button_poll_step,
  .enable = sl_simple_button_enable,
  .disable = sl_simple_button_disable,
};



const sl_button_t *sl_simple_button_array[] = {
  &sl_button_btn0, 
  &sl_button_btn1
};

const uint8_t simple_button_count = 2;

void sl_simple_button_init_instances(void)
{
  sl_button_init(&sl_button_btn0); // pass pointer to data structure from above
  sl_button_init(&sl_button_btn1); // pass pointer to data structure from above
}

void sl_simple_button_poll_instances(void)
{
  sl_button_poll_step(&sl_button_btn0);
  sl_button_poll_step(&sl_button_btn1);
}


sl_simple_button_init_instances() is referenced from : autogen/sl_event_handler.c 

void sl_driver_init(void)
{
  GPIOINT_Init();
  sl_simple_button_init_instances(); // << called here
  sl_simple_led_init_instances();
}


sl_driver_init() is referenced from : sl_system_init.c

void sl_system_init(void)
{
  sl_platform_init();
  sl_driver_init();   // << called here
  sl_service_init();
  sl_stack_init();
  sl_internal_app_init();
}

sl_system_init() is referenced from : main.c

  // Initialize Silicon Labs device, system, service(s) and protocol stack(s).
  // Note that if the kernel is present, processing task(s) will be created by
  // this call.
  sl_system_init(); // << This is the top of the call-chain for initialization.





************************************************
Following the callback IRQ handler: sli_simple_button_on_change()
************************************************

sli_simple_button_on_change() callback is defined in : sl_simple_button.c 

/***************************************************************************//**
 * An internal callback called in interrupt context whenever a button changes
 * its state. (mode - SL_SIMPLE_BUTTON_MODE_INTERRUPT)
 *
 * @note The button state is updated by this function. The application callback
 * should not update it again.
 *
 * @param[in] interrupt_no      Interrupt number (pin number)
 ******************************************************************************/
static void sli_simple_button_on_change(uint8_t interrupt_no)
{
  for (uint8_t i = 0; i < simple_button_count; i++) {
  
    sl_simple_button_context_t *ctxt = ((sl_simple_button_context_t *)sl_simple_button_array[i]->context);
    
    if ( (ctxt->pin == interrupt_no) && (ctxt->state != SL_SIMPLE_BUTTON_DISABLED) ) {
    
      ctxt->state = ((bool)GPIO_PinInGet(ctxt->port, ctxt->pin) == SL_SIMPLE_BUTTON_POLARITY);
      
      sl_button_on_change(sl_simple_button_array[i]); // << follow this 
      
      break;
    }
  }
}



sl_button_on_change() is defined in : gecko_sdk_3.2.3/.../app_button_press.c 

/***************************************************************************//**
 * This is a callback function that is invoked each time a GPIO interrupt
 * in one of the pushbutton inputs occurs.
 *
 * @param[in] handle Pointer to button instance
 *
 * @note This function is called from ISR context and therefore it is
 *       not possible to call any BGAPI functions directly. The button state
 *       of the instance is updated based on the state change. The state is
 *       updated only after button release and it depends on how long the
 *       button was pressed. The button state is handled by the main loop.
 ******************************************************************************/
void sl_button_on_change(const sl_button_t *handle)
{
  uint32_t t_diff;
  // If disabled, do nothing
  if (disabled) {
    return;
  }
  // Iterate over buttons
  for (uint8_t i = 0; i < SL_SIMPLE_BUTTON_COUNT; i++) {
  
    // If the handle is applicable
    if (SL_SIMPLE_BUTTON_INSTANCE(i) == handle) {
    
      // If button is pressed, update ticks
      if (sl_button_get_state(handle) == SL_SIMPLE_BUTTON_PRESSED) {
      
        buttons[i].timestamp = sl_sleeptimer_get_tick_count();
        
      } else if (sl_button_get_state(handle) == SL_SIMPLE_BUTTON_RELEASED) {
      
        // Check time difference
        t_diff = sl_sleeptimer_get_tick_count() - buttons[i].timestamp;
        
        // Set state flag according to the difference
        if (t_diff < sl_sleeptimer_ms_to_tick(SHORT_BUTTON_PRESS_DURATION)) {
          buttons[i].state = APP_BUTTON_PRESS_DURATION_SHORT;
        } else if (t_diff < sl_sleeptimer_ms_to_tick(MEDIUM_BUTTON_PRESS_DURATION)) {
          buttons[i].state = APP_BUTTON_PRESS_DURATION_MEDIUM;
        } else if (t_diff < sl_sleeptimer_ms_to_tick(LONG_BUTTON_PRESS_DURATION)) {
          buttons[i].state = APP_BUTTON_PRESS_DURATION_LONG;
        } else {
          buttons[i].state = APP_BUTTON_PRESS_DURATION_VERYLONG;
        }
        
      } else {
        // Disabled state
        // Do nothing
      }
      
    }//if 
    
  }//for
}



There is other code that looks at the buttons[i].state values above that results in 
calling the callback app_button_press_cb() in app.c, which calls 
sl_btmesh_change_switch_position() 



******************************************************************************
Tracing UPWARDS, how the callback app_button_press_cb() in app.c is called 
******************************************************************************

In file app_button_press.c :

/***************************************************************************//**
 * Step function for button press
 ******************************************************************************/
// *** trace this ***
void app_button_press_step(void) // <<== 
{
  // Iterate over buttons
  for (uint8_t i = 0; i < SL_SIMPLE_BUTTON_COUNT; i++) {
    // If the button is pressed
    if (buttons[i].state != APP_BUTTON_PRESS_NONE) {
      // Call callback
      app_button_press_cb(i, buttons[i].state); // <<== params are: button and duration
      // Clear state
      buttons[i].state = APP_BUTTON_PRESS_NONE;
    }
  }
}

// ignore this call for now
void button_press_from_cli(sl_cli_command_arg_t *arguments)
{
  uint8_t button_id;
  uint8_t duration;
  button_id = sl_cli_get_argument_uint8(arguments, BUTTON_ID_PARAM_IDX);
  duration = sl_cli_get_argument_uint8(arguments, DURATION_PARAM_IDX);
  if (duration > APP_BUTTON_PRESS_DURATION_VERYLONG) {
    duration = APP_BUTTON_PRESS_DURATION_VERYLONG;
  }
  if (button_id >= SL_SIMPLE_BUTTON_COUNT) {
    button_id = SL_SIMPLE_BUTTON_COUNT - 1;
  }
  app_button_press_cb(button_id, duration); // <<== params are: button and duration
}


Trace app_button_press_step() : called from file : sl_event_handler.c 

void sl_service_process_action(void)
{
  app_button_press_step(); // <<==
  sli_simple_timer_step(); 
  sl_cli_instances_tick();
}


Trace sl_service_process_action() : called from file : sl_system_process_action.c 

void sl_system_process_action(void)
{
  sl_platform_process_action();
  sl_service_process_action(); // <<==
  sl_stack_process_action();
  sl_internal_app_process_action();
}


Trace sl_system_process_action() : called from main.c ( main while (1) loop )

   sl_system_process_action(); // << This is the top of the call-chain for execution/handling 



***********************************************************
Tracing DOWNWARDS, in app_button_press_cb() 
***********************************************************

In file app.c, app_button_press_cb()

case APP_BUTTON_PRESS_DURATION_LONG:
      // Handling of button press greater than 1s and less than 5s
      if (button == BUTTON_PRESS_BUTTON_0) {
        sl_btmesh_change_switch_position(SL_BTMESH_LIGHTING_CLIENT_OFF);
      } else {
        sl_btmesh_change_switch_position(SL_BTMESH_LIGHTING_CLIENT_ON);
      }



In file sl_btmesh_lighting_client.c 

/*******************************************************************************
 * This function change the switch position and send it to the server.
 *
 * @param[in] position Defines switch position change, possible values are:
 *                       - SL_BTMESH_LIGHTING_CLIENT_OFF
 *                       - SL_BTMESH_LIGHTING_CLIENT_ON
 *                       - SL_BTMESH_LIGHTING_CLIENT_TOGGLE
 *
 ******************************************************************************/
void sl_btmesh_change_switch_position(uint8_t position)
{
  if (position != SL_BTMESH_LIGHTING_CLIENT_TOGGLE) {
    switch_pos = position;
  } else {
    switch_pos = 1 - switch_pos; // Toggle switch state
  }

  // Turns light ON or OFF, using Generic OnOff model
  if (switch_pos) {
    log(ONOFF_LIGHTING_LOGGING_ON);
    lightness_percent = LIGHTNESS_PCT_MAX;
  } else {
    log(ONOFF_LIGHTING_LOGGING_OFF);
    lightness_percent = 0;
  }
  // Request is sent 3 times to improve reliability
  onoff_request_count = ONOFF_RETRANSMISSION_COUNT;

  send_onoff_request(0);  // Send the first request

  // If there are more requests to send, start a repeating soft timer
  // to trigger retransmission of the request after 50 ms delay
  if (onoff_request_count > 0) {
    sl_status_t sc = sl_simple_timer_start(&onoff_retransmission_timer,
                                           ONOFF_RETRANSMISSION_TIMEOUT,
                                           onoff_retransmission_timer_cb,
                                           NO_CALLBACK_DATA,
                                           true);
    app_assert_status_f(sc, "Failed to start periodic timer\n");
  }
} // sl_btmesh_change_switch_position()



/***************************************************************************//**
 * This function publishes one generic on/off request to change the state
 * of light(s) in the group. Global variable switch_pos holds the latest
 * desired light state, possible values are:
 * switch_pos = 1 -> PB1 was pressed long (above 1s), turn lights on
 * switch_pos = 0 -> PB0 was pressed long (above 1s), turn lights off
 *
 * param[in] retrans  Indicates if this is the first request or a retransmission,
 *                    possible values are 0 = first request, 1 = retransmission.
 *
 * @note This application sends multiple generic on/off requests for each
 *       long button press to improve reliability.
 *       The transaction ID is not incremented in case of a retransmission.
 ******************************************************************************/
static void send_onoff_request(uint8_t retrans)
{
  struct mesh_generic_request req;
  const uint32_t transtime = 0; // using zero transition time by default
  sl_status_t sc;

  req.kind = mesh_generic_request_on_off;
  req.on_off = switch_pos ? MESH_GENERIC_ON_OFF_STATE_ON : MESH_GENERIC_ON_OFF_STATE_OFF;

  // Increment transaction ID for each request, unless it's a retransmission
  if (retrans == 0) {
    onoff_trid++;
  }

  // Delay for the request is calculated so that the last request will have
  // a zero delay and each of the previous request have delay that increases
  // in 50 ms steps. For example, when using three on/off requests
  // per button press the delays are set as 100, 50, 0 ms
  uint16_t delay = (onoff_request_count - 1) * REQ_DELAY_MS;

  sc = mesh_lib_generic_client_publish(MESH_GENERIC_ON_OFF_CLIENT_MODEL_ID,
                                       BTMESH_LIGHTING_CLIENT_MAIN,
                                       onoff_trid,
                                       &req,
                                       transtime, // transition time in ms
                                       delay,
                                       NO_FLAGS   // flags
                                       );

  if (sc == SL_STATUS_OK) {
    log_info(LIGHTING_ONOFF_LOGGING_CLIENT_PUBLISH_SUCCESS, onoff_trid, delay);
  } else {
    log_btmesh_status_f(sc, LIGHTING_ONOFF_LOGGING_CLIENT_PUBLISH_FAIL);
  }

  // Keep track of how many requests has been sent
  if (onoff_request_count > 0) {
    onoff_request_count--;
  }
  
} // send_onoff_request()


mesh_lib_generic_client_publish() is the method (function) that clients uses to publish 
a generic on/off message. 




************************************************************************
Summary of modifications to create the client-publisher starter code 
************************************************************************

All of the models remain. Some functionality from the example code has been disabled or
modified.

files = gecko_sdk_3.2.3/app/common/util/app_button_press/app_button_press.c 
        gecko_sdk_3.2.3/app/common/util/app_button_press/app_button_press.h
See my initials: DOS and "Student Edits".

Changed sl_button_on_change() to set "pressed" and "released" states.
Removed/disabled the measurement of button press code.


file = app.c 
See my initials: DOS

Modified app_button_press_cb() to only respond to button "press" and button "release"

Changed device from "switch node" to "cli-pub" for client-publisher




************************************************************************
Summary of modifications to create the server-sub-scriber starter code 
************************************************************************

All of the models remain. Some functionality from the example code has been disabled or
modified.


file = app.c 
See my initials: DOS

Changed device from "light node" to "ser-sub" for server-subscriber


file = app_out_lcd.c 
See my initials: DOS and "Student Edits".

Commented out functionality for sl_btmesh_ctl_on_ui_update(). The CTL models are still
there in the publisher, they just can't send any messages.

Modified sl_btmesh_lighting_server_on_ui_update() to have students convert the
lightness level into "Button Pressed" and "Button Released" messages on the LCD.

Left the control of the LEDs the unmodified.




