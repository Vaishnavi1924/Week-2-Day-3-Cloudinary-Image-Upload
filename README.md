# Week-2-Day-3-Cloudinary-Image-Upload
Vendor selects image         ↓ React frontend         ↓ Node/Express backend         ↓ Cloudinary         ↓ Image URL returned         ↓ MongoDB Product.image


1. Create a Cloudinary account

Go to:

Cloudinary

Create/sign in to your account.

In the Cloudinary dashboard, find:

Cloud Name
API Key
API Secret

Keep these values private.

2. Install required packages

Open the terminal:

cd backend

Run:

npm install cloudinary multer

You already installed these earlier, so npm may simply say they are already up to date.

3. Add Cloudinary configuration

Create:

backend/config/cloudinary.js

Add:

const cloudinary = require("cloudinary").v2;

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET
});

module.exports = cloudinary;

Cloudinary recommends using its Node.js SDK for backend uploads.

4. Update .env

Open:

backend/.env

Add:

CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret

So your .env becomes:

PORT=5000

MONGO_URI=your_mongodb_connection_string

JWT_SECRET=my_super_secret_jwt_key_123

CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret



5. Create upload middleware

Create:

backend/middleware/uploadMiddleware.js

Add:

const multer = require("multer");

const storage = multer.memoryStorage();

const upload = multer({
  storage: storage,
  limits: {
    fileSize: 5 * 1024 * 1024
  },
  fileFilter: (req, file, cb) => {
    if (file.mimetype.startsWith("image/")) {
      cb(null, true);
    } else {
      cb(new Error("Only image files are allowed"));
    }
  }
});

module.exports = upload;

This means:

Images only ✅
Maximum size = 5 MB
File temporarily stays in memory
We don't need to create an uploads folder
6. Create upload controller

Create:

backend/controllers/uploadController.js

Add:

const cloudinary = require("../config/cloudinary");

const uploadImage = async (req, res) => {
  try {

    if (!req.file) {
      return res.status(400).json({
        message: "Please select an image"
      });
    }

    const result = await new Promise((resolve, reject) => {

      const stream = cloudinary.uploader.upload_stream(
        {
          folder: "multi-tenant-ecommerce/products"
        },
        (error, result) => {

          if (error) {
            reject(error);
          } else {
            resolve(result);
          }

        }
      );

      stream.end(req.file.buffer);
    });

    res.status(200).json({
      message: "Image uploaded successfully",
      imageUrl: result.secure_url,
      publicId: result.public_id
    });

  } catch (error) {

    res.status(500).json({
      message: error.message
    });

  }
};

module.exports = {
  uploadImage
};

Cloudinary's Node SDK supports stream-based uploads, which is useful when your backend receives uploaded files.

7. Create upload route

Create:

backend/routes/uploadRoutes.js

Add:

const express = require("express");

const {
  uploadImage
} = require("../controllers/uploadController");

const {
  protect,
  authorizeRoles
} = require("../middleware/authMiddleware");

const upload = require("../middleware/uploadMiddleware");

const router = express.Router();

router.post(
  "/image",
  protect,
  authorizeRoles("vendor"),
  upload.single("image"),
  uploadImage
);

module.exports = router;

The important part is:

upload.single("image")

It means the frontend must send the file using the field name:

image
8. Connect upload routes

Open:

backend/server.js

Add:

const uploadRoutes = require("./routes/uploadRoutes");

Then add:

app.use("/api/upload", uploadRoutes);

Your server now has:

POST /api/upload/image
9. Test Cloudinary with Postman

Start your backend:

cd backend
npm run dev

If you don't have the dev script yet, run:

node server.js

Then open Postman.

Request:

POST
http://localhost:5000/api/upload/image

Authorization:

Bearer YOUR_VENDOR_TOKEN

Go to:

Body
→ form-data

Add:

Key	Type	Value
image	File	Select an image

Then click Send.

You should receive:

{
  "message": "Image uploaded successfully",
  "imageUrl": "https://res.cloudinary.com/...",
  "publicId": "multi-tenant-ecommerce/products/..."
}

The imageUrl is what we'll save inside MongoDB.

10. Update Product Management

Now let's modify:

frontend/src/pages/ProductManagement.jsx

First, add this state:

const [imageFile, setImageFile] = useState(null);

Then add this function:

const handleImageChange = (e) => {
  setImageFile(e.target.files[0]);
};

Inside your form, replace the old:

<input
  name="image"
  placeholder="Image URL"
  value={formData.image}
  onChange={handleChange}
/>

with:

<input
  type="file"
  accept="image/*"
  onChange={handleImageChange}
/>
11. Update handleSubmit

Replace your existing handleSubmit with:

const handleSubmit = async (e) => {
  e.preventDefault();

  try {

    let imageUrl = "";

    // Upload image first
    if (imageFile) {

      const imageData = new FormData();

      imageData.append("image", imageFile);

      const uploadResponse = await API.post(
        "/upload/image",
        imageData,
        {
          headers: {
            "Content-Type": "multipart/form-data"
          }
        }
      );

      imageUrl = uploadResponse.data.imageUrl;
    }

    // Create product
    await API.post("/products", {
      name: formData.name,
      description: formData.description,
      price: Number(formData.price),
      stock: Number(formData.stock),
      category: formData.category,
      image: imageUrl
    });

    setMessage("Product added successfully");

    setFormData({
      name: "",
      description: "",
      price: "",
      stock: "",
      category: "",
      image: ""
    });

    setImageFile(null);

    fetchProducts();

  } catch (error) {

    setMessage(
      error.response?.data?.message ||
      "Unable to add product"
    );

  }
};
12. Display Product Image

Inside:

products.map(...)

Add:

{product.image && (
  <img
    src={product.image}
    alt={product.name}
    width="150"
  />
)}

For example:

<div
  key={product._id}
  className="product-card"
>

  {product.image && (
    <img
      src={product.image}
      alt={product.name}
      width="150"
    />
  )}

  <h3>{product.name}</h3>

  <p>{product.description}</p>

  <p>
    Price: ₹{product.price}
  </p>

  <p>
    Stock: {product.stock}
  </p>

  <p>
    Category: {product.category}
  </p>

  <button
    onClick={() =>
      deleteProduct(product._id)
    }
  >
    Delete
  </button>

</div>
13. Add image CSS

Add to:

frontend/src/index.css
.product-card img {
  width: 150px;
  height: 150px;
  object-fit: cover;
  border-radius: 8px;
  margin-bottom: 10px;
}
14. Your final product flow

Now your application works like this:

                 VENDOR
                   │
                   ▼
          Product Management
                   │
           Select Product Image
                   │
                   ▼
                React
                   │
                   ▼
             Express API
                   │
                   ▼
              Multer
                   │
                   ▼
             Cloudinary
                   │
                   ▼
            Image URL
                   │
                   ▼
               MongoDB
                   │
                   ▼
             Product.image
