This repository has seen no work since 2015 (except dependency updates) and was mostly for learning and internal usage. It is now archived.

# itemRental - Node JS library system
This is a item self-checkout system, where the back end is running Node with Sequelize and the front end is implemented in Jade and makes heavy use of DataTables to display data. It was developted at Telekom Innovation Laboratory of the QU chair at the Technical University of Berlin.

## Backend

### Modules
The main app.js file only requires all necessary node plugins and delegates all tasks. Following plugins are used:
#### Express
[Express](http://expressjs.com) is used as the minimal framework to handle routing and so on. We also use the bundled [Jade](http://jade-lang.com) rendering engine.
#### Morgan
[Morgan](https://github.com/expressjs/morgan) is used for logging in dev mode.
#### cookieParser, session, bodyParser
Since Express 4, [cookieParser](https://github.com/expressjs/cookie-parser), [session](https://github.com/expressjs/session) and [bodyParser](https://github.com/expressjs/body-parser) have been exported from the framework and live on their own. The first two are used for our (simple) authentification against the username in the db, the last one provides simple parsing capabilities for request bodies in JSON or RAW.

### Setup
#### Models
All models live in /models/, where the index.js both holds the database configuration and traverses the whole folder for other *.js files to use as models. Furthermore it will automatically create an admin user in development mode. Following two models are created:
##### user
Defines an user as a set of `username`, `firstname`, `lastname`, `role` (user/admin) and `active` flag. The association is `User.hasMany(Item, {foreignKey: 'Owner'})`.

##### item
Defines an item as a set of:
* **Label** (INTEGER(4).UNSIGNED, allowNull: false) - The 4 digit label on each piece
* **Item** (TEXT, allowNull: false) - The name of an item
* **Description** (TEXT) - A general description of that item
* **Serial** (TEXT) - SerialNo if that item, if there is one
* **Category** (TEXT) - The general category of that item
* **URL** (STRING) - An URL for more information
* **Location** (TEXT) - Location if that item in the format room shelf.level, ie Pinta 2.4
* **Status** (STRING) - Availability status of the item - in / out / nocirc (currently not used)
* **Condition** (STRING) - Physical condition of that item - good / mended / broken
* **Comment** (TEXT) - Free text comment on item such as buying date, repair date, possible defects

Furthermore, **Label** has a custom getter function, since SQLite doesn't support zero-filling:
```javascript
get: function(){
  return ("000"+this.getDataValue('Label')).slice(-4);
}
```
Mirroring the association for users, an item is defined as `Item.belongsTo(models.User, {foreignKey: 'Owner'})`;

#### Routes / API
Similarly to the Models, Routes are exported from the /routes folder and are collected by the `index.js`, which also defines the three main routes:
* `GET / ` will look up the maximum label number and forward it together with the `req.session.user` to render the `index.jade` template.
* `POST /login` will look up the `req.body.username` in the table of users. If we can find someone, we check the `useLDAP` flag, if it is `false` , simply set the current `req.session.user` to that user and send a `200`. If LDAP is used, the `ldapCheck` function is forwarded the provided username and password. If any step fails, we simply abort with a `401`.
* `ALL /logout` any request to `/logout` is followed by a `req.session.destroy`and a reload.

##### users
Following user routes are defined in the `routes/user.js`, most are only accesible as an admin.
* `GET /users` returns a JSON list of all users if someone is logged in as an admin, otherwise `401` is sent.
* `POST /users` upserts (insert or updates) a user with the provided `req.body.username` and following attributes: `req.body.firstname, req.body.lastname, req.body.role, req.body.active`. Returns `200` on successfull upsertion or `401` if user sending the request is not an admin.
* `GET /users/:username` returns the information of the provided `req.param.username` to admins or `401` if user sending the request is not an admin.
* `DELETE /users/:username` deletes user with provided `req.param.username`. Returns `200` on succesfull deletion, `404` if user not found and `401` if user sending the request is not an admin.
* `GET /users/check` Returns information on logged-in user. Returns `401`if not logged in. TODO: Delete in favor of using GET /users/req.session.username
* `GET /users/checkItems` Returns borrowed items on a user. Returns `401`if not logged in.
* `GET /syncWithLDAP` special route that grabs all users from a custom as an php-serialized array, as defined in the config. There, the correct user role is also read from and applied to the user. This is probably not much use for anyone outside this very special case.

##### items
* `GET /items` Returns JSON of all items in db. Should in future probably return less data.
* `POST /items` Upserts (insert or update) item from the provided req.body data. If a `req.body.owner` is sent, it has to match an existing username. Returns `401` if not logged in as admins.
* `GET /items/:item_label` Returns item information of provided `item_label` or `404` if not found.
* `DELETE /items/:item_label` Delets a single item from the provided item_label, returns `401` if user sending the request is not an admin.
* `GET /items/:item_label/owner` Returns current owner of provided `req.param.item_label`. Returns `404` if the item doesn`t have an owner or item is not found.
* `POST /items/:item_label/owner` Sets current owner of provided `req.param.item_label`. Returns `404` if the item wasn't found and `401` if user is not logged in. TODO: Don't override other checkouts.
* `DELETE /items/:item_label/owner` Deletes the current owner of provided `req.param.item_label`. Returns `404` if item or owner wasn't found and `401` if user is not logged in as admin.
* `POST /items/createBulk` Special function that takes a stringified JSON array of objects to create them in bulk, ignoring duplicates. Used for writing data from CSV and returns `401` if user not an admin.

#### Views
Only a very small number of views is needed.
* `index.jade` provides the basic html layout and includes all other views. TODO: modularize the .js and .css includes
* `navbar.jade` defines the navbar including the searchbar and the simple greeting top
* `modals.jade` holds all four modals:
  * `#loginModal` provides username / password login
  * `#userModal` shows currently borrowed items
  * `#borrowModal` allows borrowing of available items
  * `#adminModal` includes all admin functionality (see below) - only visible to admins
* `admin.jade` provides item return, add item, user list, add user and csv bulk creation.

## Frontend
### Modules
The following 3rd party modules were used:
#### Jquery
Most button events and ajax calls are done using jquery.
#### Bootstrap
Not only the style but a lot of the js plugin functionality was used, inlcuding modals, forms, navbars and panels.
#### dataTables
DataTables was used to display most of the data returned by the API. In total, 4 different instances were used: The main item search, overview of borrowed items, list of users (admin only), parsed CSV data (admin only).
#### papaParse
PapaParse was used for parsing the CSV data into a JSON array that could be displayed in a dataTable and then send off to bulkCreate.
#### jquery.sortable
This small library was used to allow for the reordering of data columns for the CSV data.
#### intro.js
Used to display a helpful usage hints.

## Building

### Install core
1. If you have not yet install git: [GIT download page](https://git-scm.com/downloads)
1. Have node and npm installed (Either as a [binary install](https://nodejs.org/download/) or from a [package manager](https://github.com/joyent/node/wiki/Installing-Node.js-via-package-manager))
2. Install bower-move: `npm install -g bower-move`

### Build the project
1. Clone this repository or download the .zip and extract
2. Navigate with your comand line into that folder and run `npm install`. This will download all necessary backend dependencies and put them into the node_modules folder
2. Afterwards, run `bower-move`. This will download all front-end dependencies into the bower_components folder and move the compiled files into the correct /public/ subfolder. You can also pass it the  `--delete` flag, in order to delete the bower_components folder after moving.

### Optional: Themeing
If you want to re-theme, you need to install the css-preprocessor LESS to re-compile bootstrap_flatly_custom.min.css
1. Install less: `npm install -g less`, if you want to build your own theme
2. Recheck if bower-move has moved all bootstrap .less files from 'bower_components/bootstrap/less/*' to 'src/boostrap_style'.
2. Inside 'src/boostrap_style', run `lessc -x bootswatch.less > ../../public/custom/bootstrap_flatly_custom.min.css` to compile the .less files.
4. Run `ls -1 | grep -E -v 'variables.less|bootswatch.less' | xargs rm -rf` to delete all files but bootswatch.less and variables.less

## Set up server
To make it run on port 80, you first need to disable apache2 and then link port 3000 to port 80. The following has only been tried on Ubuntu.

1. Run the following in your command line to disable apache2 (if running): `sudo service apache2 stop`, or to stop autoboot: `sudo update-rc.d -f apache2 remove`
2. Link port 3000 -> port 80:
 ```
 sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 3000
 ````
3. To do this on boot, add the following line in `/etc/rc.local`: 
 ```
 iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 3000
 ```
 
4. Install forever for restart on errors and logs: `sudo npm install -g forever`
5) To add forever as upstart script, add the file `/etc/init/libnode.conf` with the following content:

```
description "Upstart libnode server with forever"
# make sure we can write log files and start servers
start on filesystem and started networking
stop on shutdown
env NODE_ENV=production
env PORT=3000
script
  exec forever start ~/node/libnode/bin/www
end script
pre-stop script
  exec forever stopall
end script
```