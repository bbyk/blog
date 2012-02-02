## Domain Model vs View Model ##

What do you think about [separation of concerns (SoC)](http://en.wikipedia.org/wiki/Separation_of_concerns)? Isn't it that OOP was invented for? At least to alleviate the process.

Though mostly all modern languages have OOP stuff in it, developers often tend to use objects in a functional way. It is common thing nowadays that you can see a lot of classes working essentially as utils. They take data in arguments of its methods and change them. This is what is often called [anemic domain model](http://martinfowler.com/bliki/AnemicDomainModel.html).

I prefer to follow another paradigm - **rich domain model** and below in the article I will illustrate the idea. Besides I will show difference between Doman Model and View Model when dealing with UI.

### Domain Model ###

Ok, here we go. Imagine you have to implement UI that changes names of files. To see what I am talking about, open your favorite file manager, pick a file, right click on it and select "Rename". You will probably see either a dialog or in-place editor to change name of the file.

Let's consider what is domain model here. As domain model is about rules, calculations etc, I would say it is all about rules that name of a file should comply with.

	public class FileDomain {
		private static final String RESERVED_CHARS = ":*/<>?\|";
		public static final int MAX_LEN = 256;
		
		public boolean isCharAllowed(char c) {
			boolean foundInReserved = RESERVED_CHARS.indexOf(c) > -1;
			return foundInReserved;
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

This piece of code is authoriative for any rules about file names. And we have to consult it from any other subsystem. Though the example works with just a static read-only state, it uses on OOP paradigm - encapsulation. There can be also non-static state to encapsulate about file names:

	public class FileDomain {
		...
		File file;		
		...

		public String getName() {
			return file.getName();
		}

		public void setName(@NotNull String newName) {
			assert newName != null;

			if (isNameCorrect(newName))
				throw new IllegalArgumentException("incorrect name of file: " + newName);

			file.setName(newName);
		}
			
    ....
	}

What's the File? In this example File is another piece of functionality that you want to separate from logic. File represents operations on your file-system. That can be OS backed filesystem or you can maintain your own one that you store in a DB. In the model you're separated from it through the interface:

	public interface File {
		...
		String getName();
		void setName(String value);
		...
	}

This is actually a [Bridge pattern](http://en.wikipedia.org/wiki/Bridge_pattern "Bridge pattern") employed here. It is convenient to use if you plan to write unit-tests for your domain objects. What you have to do in this case is to create a mocked implementation of File and set it into your instance of FileDomain:

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


Our domain model class encapsulates operations on its static and non-static state. The former one is represented by instance of File. If you get the hang of it, let's consider how View Model would look like.

### View Model ###

UI deals with different domain. From UI perspective the domain is different. Ok not completly different but, say, extended. It's mostly about how it looks and feels rather than be persisted.

What you see in UI is essentially a UI component that has 2 modes. It can either render the file name or edit it. Then, being in edit mode it keeps track of text entered, selection, copy/paste etc.

All this stuff is going to land into your View Model that essentially represents your domain logic for UI. Let's take a look at the state your View model will manage. I will skip selection for simplicity sake:

	public class ViewDomain {
		private bool isInEditMode;
		private FileDomain fileDomain;
		private StringBuilder temporaryFileName;

		...
	}

It is time to mention that I am going [MVC](http://en.wikipedia.org/wiki/Model–view–controller "MVC pattern") way on the UI side. That assumes I will use my model from a controller that reactes on events from one or more views. Suppose user presses a key and that is the event that will be eventually handled in the controller. And the controller has to consult model if the character entered is blessed. If so, we have to append it to the *temporaryFileName*

	public class ViewDomain {
		...

		public bool isInEditMode() {
			return isInEditMode;
		}

		public void startEdit() {
			isInEditMode = true;
			temporaryFileName.setLength(0);
			temporaryFileName.append(fileDomain.getName());

			fireSwitchedToEditModeEvent();
		}

		public String getTemporaryFileName() {
			return temporaryFileName.toString();
		}

		public void appendChar(char c) throws CharacterNotAllowedException {
			if (!fileDomain.isLengthAllowed(temporaryFileName.length() + 1);
				fireOutOfLengthAllowed(FileDomain.MAX_LEN);

			if (!fileDomain.isCharAllowed(c))
				fireCharNotAllowedEvent(c);

			temporaryFileName.append(c);
		}

		public void acceptEdit() {
			try {
				fileDomain.setName(temporaryFileName.toString());
				stopEditMode();
			} catch (BadFileNameException e) {
				// still we can receive this if model rules are changed
				fireBadFileNameEvent(e)
			}
		}

		public void stopEditMode() {
			isInEditMode = false;
			temporaryFileName.setLength(0);

			fireSwitchedToReadModeEvent();
		}
		...
	}

You might have noticed that the View Model tends to fire event rather than throw exception. That's according to MVC paradigm. View listens to the model and **had to be notified** on any change in state that just happened. Upon being notified the view will query the model and render a new view or show errors which are also a part of view.

Again we see that the View model employes one of the base OOP paradigm: encapsulation. And View Model delegates some behavior to File Domain model aggregating the instance of the object.

Now let's say you want to put the all stuff together:

	public class FileNameEditorComponent {
		private Controller controller;
		private View view;
		private ViewModel model;

		...
		public void FileNameEditorComponent() {
			view.addKeyPressedListner(controller.onKeyPressed);
			...
			model.addSwitchedToEditModeEventListner(view.onSwitchedToEditModeEvent);
			model.addSwitchedToReadModeEventListner(view.onSwitchedToReadModeEvent);
			model.addCharNotAllowedEventListner(view.onCharNotAllowedEvent);
			...
		}
		...
	}

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
				// check if c is controll key and call different model methods
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

I intentionally did not use any specific UI MVC framework to build up the set of examples. My goal was to demostrate a concepts and how the pieces, how the pieces of concern a separated so each of them was responsible only for its own functionality.

Besides all models FileDomain and View model in the example can be easily covered with unit-tests. I guess some controller functionality can be covered too.

Another benefit of the granularity is the ability to put mocked view implementation and run subset of integration tests.