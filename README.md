# REALMF
This Python code utilizes the OpenCV and Face++ API to capture video from a webcam, detect faces, and classify them based on gender, age, and other attributes.

import cv2
import requests
import base64

# Replace with your Face++ API key and secret
api_key = 'Replace with your Face++ API key'
api_secret = 'Replace with your Face++ Secret key'
# Face++ API endpoint for detecting faces
detect_url = 'https://api-us.faceplusplus.com/facepp/v3/detect'
params = {}

capture_pressed = False


def detect_faces(image):
    # Resize the image to a smaller size
    resized_img = cv2.resize(image, (800, 600))

    # Convert the resized image to RGB format
    img_rgb = cv2.cvtColor(resized_img, cv2.COLOR_BGR2RGB)

    # Encode the resized image as base64 for the API request
    _, img_encoded = cv2.imencode('.png', img_rgb)
    img_base64 = base64.b64encode(img_encoded).decode('utf-8')

    # API request parameters
    params = {
        'api_key': api_key,
        'api_secret': api_secret,
        'image_base64': img_base64,
        'return_attributes': 'gender,age'
    }

    # Make API request to detect faces
    response = requests.post(detect_url, data=params)

    # Check if the response status code is 200 (OK)
    if response.status_code == 200:
        try:
            data = response.json()
            print('API Response:', data)  # Print the entire response for debugging
            return data.get('faces', [])
        except requests.exceptions.JSONDecodeError:
            print('Error: Empty or invalid JSON response from the Face++ API.')
            return []
    else:
        print(f'Error: Non-OK status code from Face++ API - {response.status_code}')
        return []


def draw_face_rectangles(image, faces, gender_to_display=None, print_age=False):
    for face in faces:
        # Get face rectangle coordinates from the response
        face_rectangle = face.get('face_rectangle', {})
        x, y, w, h = face_rectangle.get('left', 0), face_rectangle.get('top', 0), face_rectangle.get('width', 0), face_rectangle.get('height', 0)

        # Get gender information from the response
        gender_info = face.get('attributes', {}).get('gender', {})
        gender = gender_info.get('value', 'unknown').lower()

        # Filter faces based on gender_to_display
        if gender_to_display is not None and gender != gender_to_display:
            continue

        # Draw rectangles and labels with reduced size
        rectangle_thickness = 1
        cv2.rectangle(image, (x, y), (x + w, y + h), (255, 0, 0), rectangle_thickness)

        # Get gender and age information
        age_info = face.get('attributes', {}).get('age', {})
        age = age_info.get('value', 0)

        # Reduce the font size for labels
        font_size = 0.5
        cv2.putText(image, f'Gender: {gender}, Age: {age}', (x, y - 5), cv2.FONT_HERSHEY_SIMPLEX, font_size, (0, 255, 0), rectangle_thickness)

        # Print the age if requested
        if print_age:
            print(f'Person detected. Age: {age}')



def classify_faces(image):
    # Detect faces using Face++ API
    faces = detect_faces(image)

    # Initialize counts
    total_faces, male_count, female_count, child_count, odd_count, low_quality_count = count_faces(faces)

    # Draw rectangles and labels with age information
    draw_face_rectangles(image, faces, print_age=True)

    # Print counts
    print_counts(total_faces, male_count, female_count, child_count, odd_count)

    return image


def count_faces(faces):
    total_faces = len(faces)
    male_count = 0
    female_count = 0
    child_count = 0
    odd_count = 0
    low_quality_count = 0

    for face in faces:
        # Get gender information from the response
        gender_info = face.get('attributes', {}).get('gender', {})
        gender = gender_info.get('value', 'unknown').lower()

        # Get age information from the response
        age_info = face.get('attributes', {}).get('age', {})
        age = age_info.get('value', 0)

        # Increment counts based on gender
        if gender == 'male':
            male_count += 1
        elif gender == 'female':
            female_count += 1

        # Increment child count
        if age < 18:
            child_count += 1

        # Increment odd count based on your criteria (e.g., glasses)
        glasses_info = face.get('attributes', {}).get('glasses', {})
        glasses = glasses_info.get('value', 'None').lower()
        if glasses != 'normal':
            odd_count += 1

        # Increment low-quality count based on confidence score
        confidence = face.get('face_probability', 0)
        if confidence < 0.8:
            low_quality_count += 1

    return total_faces, male_count, female_count, child_count, odd_count, low_quality_count


def print_counts(total_faces, male_count, female_count, child_count, odd_count, age=None):
    print('Total number of faces:', total_faces)
    print('Number of male faces:', male_count)
    print('Number of female faces:', female_count)
    print('Number of child faces:', child_count)
    # print('Number of odd faces:', odd_count)

    if age is not None:
        print('Age of the Person:', age)


# Initialize the webcam
cap = cv2.VideoCapture(0)  # Use 0 for the default webcam

# Capture an image from the webcam
ret, frame = cap.read()

# Classify faces in the captured image
processed_frame = classify_faces(frame)

# Get screen width and height
screen_width = 1920  # Set your screen width
screen_height = 1080  # Set your screen height

# Resize the image to fit the screen
processed_frame = cv2.resize(processed_frame, (screen_width, screen_height))

# Display the image in full screen
cv2.namedWindow('Detected Faces with Attributes', cv2.WND_PROP_FULLSCREEN)
cv2.setWindowProperty('Detected Faces with Attributes', cv2.WND_PROP_FULLSCREEN, cv2.WINDOW_FULLSCREEN)

cv2.imshow('Detected Faces with Attributes', processed_frame)

# Wait for a key press and then release the webcam
cv2.waitKey(0)
cap.release()
cv2.destroyAllWindows()
