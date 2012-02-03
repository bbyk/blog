## Domain Model vs View Model ##

What do you think about [separation of concerns (SoC)](http://en.wikipedia.org/wiki/Separation_of_concerns)? Isn't it that OOP was invented for? At least to alleviate the process.

Although most modern programming languages have OOP stuff in it, developers often tend to use objects in a functional way. It is common thing nowadays that you can see a lot of classes working essentially as utilities. They take data in arguments of their methods and change them. This is what is often called [anemic domain model](http://martinfowler.com/bliki/AnemicDomainModel.html).

I prefer to follow another paradigm - **rich domain model** and below in the article I will illustrate the idea. Besides I will show difference between Doman Model and View Model when dealing with UI.

### Domain Model ###

Here we go. Imagine you have to implement UI that changes names of files. To see what I am talking about, open your favorite file manager, pick a file, right click on it and select "Rename". You will probably see either a dialog or an in-place editor to change name of the file.

Let's consider what is domain model here. As domain model is all about rules, calculations etc, I would say that it is all about rules which a file name should comply with.

	public class FileDomain {
		private static final String RESERVED_CHARS = ":*/<>?\|";
		public static final int MAX_LEN = 256;
		
		public boolean isCharAllowed(char c) {
			boolean foundInReserved = RESERVED_CHARS.indexOf(c) > -1;
			return !foundInReserved;
		}

		public boolean isLengthAllowed(int len) {
			boolean isWithinAllowedRange = len >= 0 && len <= MAX_LEN;
			return isWithinAllowedRange;
		}

		public boolean isNameCorrect(@NotNull String name) {
			assert name != null;

			if (!isLengthAllowed(name.length()))
				return false;
			for (char c : name.toCharArray()) {
				if (isCharAllowed(c))
					return false;
			}

			return true;
		}
    ....
	}

This piece of code is authoriative for any rules about file names. And we have to consult it from any other subsystem. Though the example works with just a static read-only state, it uses an OOP paradigm - encapsulation. There can be also non-static state to encapsulate:

	public class FileDomain {
		...
		File file;		
		...

		public String getName() {
			return file.getName();
		}

		public void setName(@NotNull String newName) throws BadFileNameException {
			assert newName != null;

			if (isNameCorrect(newName))
				throw new BadFileNameException(newName);

			file.setName(newName);
		}
			
    	...
	}

What's the File here? In this example File is another piece of functionality that you want to separate from the logic. File represents operations on your file-system. That can be an OS backed filesystem or you can maintain your own one that you decided to store in DB. In the model implementation you are separated from this specific through the interface:

	public interface File {
		...
		String getName();
		void setName(String value);
		...
	}

This is actually a [bridge pattern](http://en.wikipedia.org/wiki/Bridge_pattern "Bridge pattern") employed here. It is convenient to use the pattern if you plan to write unit-tests for your domain objects. What you have to do in this case is to create a mocked implementation of the File and set it into your instance of FileDomain:

	FileDomain fileDomain = new FileDomain();
	File file = EasyMock.createStrictMock(File.class);
	fileDomain.file = file;

	//record expectations for file

On other hand you could use an inheritor of FileDomain that would override an abstract method to save file names:

	public class FileDomain {
		...
		public void setName(@NotNull String newName) throws BadFileNameException {
			assert newName != null;

			if (isNameCorrect(newName))
				throw new BadFileNameException(newName);

			persistName(newName);
		}

		protected abstract void persistName(String newName);
    	...
	}

	public class FileSystemFile extends FileDomain {
		...
		@Override
		protected void persistName(String newName) {
			...
		}
	}


Our domain model class encapsulates operations on its static and non-static state. The former one is represented by instance of File.

Now if you get the hang of it, let's consider how View Model would look like.

### View Model ###

From UI perspective the domain is different. Ok not completly different but, say, extended. It's mostly about how it looks and feels rather than be persisted.

What you can see in UI here is essentially a UI component that has 2 modes. It can either render the file name or edit it. Besides being in edit mode it keeps track of text entered, selection, copy/paste etc.

All the stuff is going to be controled by your View Model that basically represents your domain logic for this piece of UI. Let's take a look at the state that your View model will maintain. For simplicity sake I will skip selection though:

	public class ViewDomain {
		private bool isInEditMode;
		private FileDomain fileDomain;
		private StringBuilder temporaryFileName;

		...
	}

It is time to mention that I am going [MVC](http://en.wikipedia.org/wiki/Model–view–controller "MVC pattern") way on the UI side. That assumes I will use my model from a controller that reacts on events from one or more views.

Suppose user presses a key and that is an event that will be eventually handled in the controller. And the controller has to consult my model. If in read mode and 'e' is presssed it will ask model to switch to edit mode. By contrast, if in edit mode, controller will ask model to append the character to the *temporaryFileName* if allowed character is provided or length is within the range:

	public class ViewDomain {
		...
		public bool isInEditMode() {
			return isInEditMode;
		}

		public void startEditing() {
			isInEditMode = true;
			temporaryFileName.setLength(0);
			temporaryFileName.append(fileDomain.getName());

			fireSwitchedToEditModeEvent();
		}

		public String getTemporaryFileName() {
			return temporaryFileName.toString();
		}

		public void appendChar(char c) {
			if (!fileDomain.isLengthAllowed(temporaryFileName.length() + 1);
				fireOutOfLengthAllowed(FileDomain.MAX_LEN);

			if (!fileDomain.isCharAllowed(c))
				fireCharNotAllowedEvent(c);

			temporaryFileName.append(c);
		}

		public void acceptEdit() {
			try {
				fileDomain.setName(temporaryFileName.toString());
				stopEditing();
			} catch (BadFileNameException e) {
				// still we can receive this if model rules are changed
				fireBadFileNameEvent(e)
			}
		}

		public void stopEditing() {
			isInEditMode = false;
			temporaryFileName.setLength(0);

			fireSwitchedToReadModeEvent();
		}
		...
	}

You might have noticed that the View Model tends to fire events rather than throw exceptions. That's according to MVC paradigm. View listens to Model and **had to be notified** about any change in state that has just happened. Upon being notified View will query the model and render a new look or show errors which are also a part of view.

Again we see that the *ViewModel* employes one of the base OOP paradigm: encapsulation. And the *ViewModel* delegates some behavior to *FileDomain* aggregating the instance of the object.

Now let's say you want to put all the stuff together:

	public class Controller {
		private ViewModel model;

		public Controller(@NotNull ViewModel model, @NotNull View view) {
			assert model != null;
			assert view != null;

			this.model = model;
			view.addKeyPressedListner(onKeyPressed);
		}		

		public void onKeyPressed(char c) {
			if (model.isInEditMode()) {
				// check if c is a controll key and call different model methods
				...
				model.append(c);
			} else if (c == 'e') // e - control key to start editing
				model.startEdit();
		}
		...
	}

	public class View {
		private ViewModel model;

		public View(@NotNull ViewModel model) {
			assert model != null;

			this.model = model;

			model.addSwitchedToEditModeEventListner(onSwitchedToEditMode);
			model.addSwitchedToReadModeEventListner(onSwitchedToReadMode);
			model.addCharNotAllowedEventListner(onCharNotAllowed);			
		}
		...
		public void onCharNotAllowed(char c) {
			showErrorPopup("The char entered is forbidden: " + c);
		}
		...
	}

I intentionally did not use any specific UI MVC framework to build up the examples. My goal was to demostrate concepts and how the pieces of concern are separated so each of them was responsible only for its own functionality.

Besides the both *FileDomain* and *ViewModel* in the examples can be easily covered with unit-tests. And I guess some functionality of controllers can be covered too.

Another benefit of the granularity is ability to put a mocked view implementation and run subset of integration tests for UI.

### Summary ###

View Model can be considered as a subset of Domain Model that encloses UI specific concerns.

Our *FileDomain* class addresses just persistence concerns, rules and core behavior. From the *FileDomain* perspective the class is agnostic of ways it will be eventually used.

The *ViewModel* operates on UI-centric state and can control intermediate state and behavior that is not related to file name logic directly, e.g. selection.

It is possible to implement View Model class as an inheritor of Domain Model class or encapsulate Domain Model class inside View Model class.