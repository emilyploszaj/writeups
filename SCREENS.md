# Screens handlers and screens
The Minecraft in-game gui system is a really important system for interfacing with the server from the client. However, it is often very unapproachable to people unfamiliar to it, and there exists few resources for navigating the systems. The purpose of this document is, thus, to adequately and comprehensively explain how these systems work, and how to make use of them.

### What are screen handlers and screens?
The in-game gui system has two primary parts, those being screen handlers and screens. Screen handlers are a representation of synced state between the client and server, responsible for keeping track of and syncing basic information. Primarily, screen handlers are responsible for handling and transmitting slot interaction, as well as syncing slot updates not caused by interaction. Screens are a client side visual representation of a screen handler state, responsible for drawing the background, slots, and presenting all screen handler information in a way it is reasonable for the player.

### What state information can screen handlers manage?
The basic data a screen handler manages is slot information. The server will notify the client of any changes that happen to the item stacks stored by slots, such as furnace output changing, and the client will inform the server when the user interacts with slots, be that pressing a mouse button or interaction key (such as F to swap item with offhand or Q to toss an item), causing the client and server to both perform the action and make sure they get the same result (to prevent desyncs, which, if detected, would prompt the server to resync the entire state to rectify). These actions are abstracted away from actual button presses into actions (such as `PICKUP`, `QUICK_MOVE`, or `THROW`), as the server is not aware of the player's key bindings.

In addition to managing slots, screen handlers can also send limited data automatically which can be used for rendering or server-side logic. Screen handlers are able to create an arbitrary amount of properties to sync information from the server to the client. A property is a representation of an integer, with built in utilities for keeping track of changes used by the screen handler. When a screen handler on the server detects a change in a property, it will send the change to the client. It is important to note that this relationship is one way, and that changing a property on a client will not have it reflected on the server. There is also a tool called a property delegate that can be used to manage several properties in one, for ease of use. This is commonly used by block entities.

Example: furnaces have 4 properties in a property delegate that are sent to the client so it can render information about the block entity. In order, it sends burn time, fuel time, cook time, and total cook time. These are used to draw the progress arrow and fuel icon.

There are situations when a client wants to send information to the server using a screen handler, and there is a utility for this. Screen handlers have a method called `onButtonClick` which is a bit deceptively named. The client has the ability (in screen handlers, screens, or elsewhere) to signify that they are clicking a button with an arbitrary index. This can be done manually using `MinecraftClient.getInstance().interactionManager.clickButton(syncId, buttonId);` where `syncId` is the screen handler's sync id and the button id is an arbitrary integer representing the button clicked. The screen handler on the server side can process this integer however it wants. For example, the stonecutter has a variety of recipes the player can choose from. In order to inform the server when the client picks one of these options, the client sends a button click being the index of the recipe it wants, and then the server updates the output if applicable (it interestingly also applies this to a property, so that the client can render which option is picked, instead of just keeping track of this information on the client).

Finally, screen handlers can send an arbitrary `PacketByteBuf` from the server to the client when opened, which can be read on the client to parse any initial information needed.

For any more complicated state syncing, custom packets are required.

### That's all well in good... but how do I use them?
More of a fan of examples? Then here is an example of a basic gui system for a processing block that takes 1 input and 1 output, and displays its progress.

```java
public class ProcessorScreenHandler extends ScreenHandler {
    public PropertyDelegate propertyDelegate;

    // Client constructor
    public ProcessorScreenHandler(int syncId, PlayerInventory playerInventory) {
        // The client initially keeps track of blank inventories and property delegates
        // These are updated by the server immediately
        this(syncId, playerInventory, new SimpleInventory(2), new ArrayPropertyDelegate(2));
    }

    // Common constructor
    public ProcessorScreenHandler(int syncHandler, PlayerInventory playerInventory, Inventory inventory, PropertyDelegate propertyDelegate) {
        super(ExampleModScreenHandlers.PROCESSOR, syncId);
        this.propertyDelegate = propertyDelegate;

        // Redundancy checks
        checkSize(inventory, 2);
        checkDataCount(propertyDelegate, 2);

        // Add the properties in the property delegate to the tracked list
        this.addProperties(propertyDelegate);

        // Adding player inventory slots
        // No, there is no utility method to do this, it's copy pasted in every single screen handler
        // Note: vanilla inventories typically add hotbar slots last, see later comment
        for (int x = 0; x < 9; x++) {
            this.addSlot(new Slot(playerInventory, x, x * 18 + 8, 142));
        }
        for (int y = 0; y < 3; y++) {
            for (int x = 0; x < 9; x++) {
                this.addSlot(new Slot(playerInventory, y * 9 + x + 9, x * 18 + 8, y * 18 + 84));
            }
        }

        // Note: vanilla inventories typically put player inventory slots last, but this makes little sense from an implementation standpoint for quick moving (shift clicking) and adding player slots first can make setting a quick move implementation up less of a hassle
        // Add slots from the block entity's inventory
        this.addSlot(new Slot(inventory, 0, 56, 35));
        // There is no generic output slot, most screen handlers that need specialized slots will do something like this
        this.addSlot(new Slot(inventory, 1, 116, 35) {
            public boolean canInsert(ItemStack stack) {
                return false;
            }
        });
    }

    // Quick move (shift click) logic
    // This is one of the consistently messiest methods in the game
    // Most people rely on copy/paste from vanilla screen handlers for this logic
    // Provided a slot index that was clicked on and returns the stack moved
    // Do note that not implementing this method in a screen handler will cause the game to enter an infinite loop when a shift click occurs
    @Override
    public ItemStack transferSlot(PlayerEntity player, int index) {
        Slot slot = this.slots.get(index);
        if (slot != null && slot.hasStack()) {
            ItemStack stack = slot.getStack();
            ItemStack old = stack.copy();
            if (index < 36) { // From player inventory
                if (stack.isOf(Items.IRON_INGOT)) { // Arbitrary input check
                    if (!this.insertItem(stack, 36, 37, false)) {
                        return ItemStack.EMPTY;
                    }
                } else if (index < 9) { // From hotbar
                    if (!this.insertItem(stack, 9, 36, false)) {
                        return ItemStack.EMPTY;
                    }
                } else {
                    if (!this.insertItem(stack, 0, 9, false) {
                        return ItemStack.EMPTY;
                    }
                }
            } else { // From processor slots
                if (!this.insertItem(stack, 0, 36, true) {
                    return ItemStack.EMPTY;
                }
            }
            // Update clicked slot
            if (stack.isEmpty()) {
                slot.setStack(ItemStack.EMPTY);
            } else {
                slot.markDirty();
            }
            // Return if no change occured
            if (stack.getCount() == old.getCount()) {
                return ItemStack.EMPTY;
            }
            slot.onTakeItem(player, stack);
            return old;
        }
        return ItemStack.EMPTY;
    }
}
```

The screen:
```java
public class ProcessorScreen extends HandledScreen<ProcessorScreenHandler> {
    private static final Identifier BACKGROUND = new Identifier("example", "textures/gui/container/processor.png");

    public ProcessorScreen(ProcessorScreenHandler handler, PlayerInventory inventory, Text title) {
        super(handler, inventory, title)
    }

    @Override
	public void render(MatrixStack matrices, int mouseX, int mouseY, float delta) {
		this.renderBackground(matrices);
		super.render(matrices, mouseX, mouseY, delta);
		this.drawMouseoverTooltip(matrices, mouseX, mouseY);
	}

    @Override
    public void drawBackground(MatrixStack matrices, float delta, int mouseX, int mouseY) {
        // Note: these are 1.17 texture binding methods
        // Use RenderSystem.bindTexture() on 1.16 and earlier
        RenderSystem.setShader(GameRenderer::getPositionTexShader);
        RenderSystem.setShaderColor(1f, 1f, 1f, 1f);
        RenderSystem.setShaderTexture(0, BACKGROUND);
		int x = (this.width - this.backgroundWidth) / 2;
		int y = (this.height - this.backgroundHeight) / 2;
        this.drawTexture(matrices, x, y, this.backgroundWidth, this.backgroundHeight);
        PropertyDelegate delegate = this.handler.propertyDelegate;
        int progress = delegate.get(0) * 24 / delegate.get(1);
        // Draw progress arrow based on information from the property delegate
        // This call assumes it's on the same spot on the texture as the vanilla furnace
        this.drawTexture(matrices, x + 79, y + 34, 176, 14, progress + 1, 16);
    }
}
```

And of course no screen would be complete without registry.
Common:
```java
public class ExampleModScreenHandlers {
    public static final ScreenHandlerType<ProcessorScreenHandler> PROCESSOR = register("processor", ProcessorScreenHandler::new);

    public static void init() {
    }

    // Note that this is a using the "simple" screen handler registry
    // Some screen handlers might want to send a bit of arbitrary info on creation
    // They would use the `registerExtended` method
	public static <T extends ScreenHandler> ScreenHandlerType<T> register(String name, BiFunction<Integer, PlayerInventory, T> supplier) {
		return ScreenHandlerRegistry.registerSimple(new Identifier("example", name), new ScreenHandlerRegistry.SimpleClientHandlerFactory<T>() {
				public T create(int syncId, PlayerInventory inventory) {
					return supplier.apply(syncId, inventory);
				}
		});
	}
}
```

Client side is really simple, you just add a call in your client initializer, or if you're like me you throw it in its own registry class.
```java
public class ExampleModScreens {

    public static void init() {
        ScreenRegistry.register(ExampleModScreenHandlers.PROCESSOR, ProcessorScreen::new);
    }
}
```

And finally, you need a way to open a screen, since the assumption is this is a processing block, it'll be in the `onUse` method
```java
// In your processor block class
private static final TranslatableText TITLE = new TranslatableText("container.example.processor");
public ActionResult onUse(BlockState state, World world, BlockPos pos, PlayerEntity player, Hand hand, BlockHitResult hit) {
    // Screen opening is handled server side and the game will manage creating and setting up the screen handler client side
    if (!world.isClient) {
        player.openHandledScreen(new SimpleNamedScreenHandlerFactory((syncId, playerInventory, player) -> {
            ProcessorBlockEntity pbe = (ProcessorBlockEntity) world.getBlockEntity(pos);
            return new ProcessorScreenHandler(syncId, playerInventory, pbe, pbe.propertyDelegate);
        }, TITLE));
    }
    return ActionResult.SUCCESS;
}
```
```java
// In your processor block entity initializer
this.propertyDelegate = new PropertyDelegate() {
    public int get(int index) {
        if (index == 0) {
            return this.processTime;
        } else {
            return this.processTimeTotal;
        }
    }

    public int set(int index, int value) {
        if (index == 0) {
            this.processTime = value;
        } else {
            this.processTimeTotal = value;
        }
    }
}
```

And there you go! A fully functioning screen. Well, you're responsible for providing the textures and hooking up the registry methods as well as the block and block entity, but those are out of scope for an example.
