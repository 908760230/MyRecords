代码：

#include <iostream>
#include <algorithm>
using namespace std;

struct AVLNode
{
	int data;
	int height;
	AVLNode *left;
	AVLNode *right;
	AVLNode() :data(0), height(0), left(nullptr), right(nullptr) {};
	AVLNode(int d,int h=0,AVLNode *l=nullptr,AVLNode *r = nullptr) :data(d), height(h), left(l), right(r) {};
};

class AVLTree 
{
public:
	AVLTree() :root(nullptr) {}
	~AVLTree() { destroy(root); }
	int Height() { return height(root); }
	bool Insert(int value) { return insert(root, value); }
	void PrintBinTree() { PrintBinTree(root); }
	bool Remove(int value) { return remove(root, value); }
private:
	//求取高度
	int height(AVLNode *cur) {
		if (!cur) return 0;
		int i = height(cur->left);
		int j = height(cur->right);
		return i > j ? i + 1 : j + 1;
	}
	//在右子树插入右孩子导致AVL失衡时，我们需要进行单左旋调整
	AVLNode *LeftRotation(AVLNode *cur) {
		AVLNode* tmp = cur->right;
		cur->right = tmp->left;
		tmp->left = cur;
		cur->height = max(height(cur->left), height(cur->right)) + 1;
		tmp->height = max(height(tmp->left), height(tmp->right)) + 1;
		return tmp;
	}
	//在左子树上插入左孩子导致AVL树失衡，我们需要进行单右旋调整
	AVLNode *RightRotation(AVLNode *cur) {
		AVLNode* tmp = cur->left;
		cur->left = tmp->right;
		tmp->right = cur;
		cur->height = max(height(cur->left), height(cur->right)) + 1;
		tmp->height = max(height(tmp->left), height(tmp->right)) + 1;
		return tmp;
	}
	//在右子树上插入左孩子导致AVL树失衡,此时我们需要进行先右旋后左旋的调整
	AVLNode *RightLeftRotation(AVLNode *cur) {
		cur->right = RightRotation(cur->right);
		return LeftRotation(cur);
	}
	//在左子树上插入右孩子导致AVL树失衡,此时我们需要进行先左旋后右旋的调整
	AVLNode *LeftRightRotation(AVLNode *cur) {
		cur->left = LeftRotation(cur->left);
		return LeftRotation(cur);
	}
	AVLNode *insert(AVLNode *&cur,int value) {
		if (!cur) cur = new AVLNode(value);
		else if (value > cur->data) {
			cur->right = insert(cur->right, value);
			if (height(cur->right)-height(cur->left)==2)
			{
				if (value > cur->right->data)
					cur = LeftRotation(cur);
				else if (value < cur->right->data)
					cur = RightLeftRotation(cur);
			}
		}
		else if (value < cur->data) {
			cur->left = insert(cur->left, value);
			if (height(cur->left)-height(cur->right)==2)
			{
				if (value < cur->left->data)
					cur = RightRotation(cur);
				else if (value > cur->left->data)
					cur = LeftRightRotation(cur);
			}
		}
		cur->height = max(height(cur->left), height(cur->right));
		return cur;
	}
	AVLNode *remove(AVLNode *cur, int value) {
		if (!cur) return nullptr;
		if (value == cur->data) {
			if (cur->left && cur->right)
			{
				if (height(cur->left)>height(cur->right))
				{
					AVLNode *tmp = maximum(cur->left);
					cur->data = tmp->data;
					cur->left = remove(cur->left, tmp->data);
				}
				else {
					AVLNode *tmp = minimum(cur->right);
					cur->data = tmp->data;
					cur->right = remove(cur->right, tmp->data);
				}
			}
			else {
				AVLNode *tmp = cur;
				if (cur->left) cur = cur->left;
				else if (cur->right) cur = cur->right;
				delete tmp;
				return nullptr;
			}
		}
		else if (value > cur->data) {
			cur->right = remove(cur->right, value);
			if (height(cur->left) - height(cur->right) == 2)
			{
				if (height(cur->left->right) > height(cur->left->left))
					cur = LeftRightRotation(cur);
				else cur = RightRotation(cur);
			}
		}
		else if (value < cur->data) {
			cur->left = remove(cur->left, value);
			if (height(cur->right) - height(cur->left) == 2)
			{
				if (height(cur->right->left) > height(cur->right->right))
					cur = RightLeftRotation(cur);
				else cur = LeftRotation(cur);
			}
		}
		return cur;
	}
	AVLNode *maximum(AVLNode *cur) {
		if (!cur) return nullptr;
		while (cur->right)
		{
			cur = cur->right;
		}
		return cur;
	}
	AVLNode *minimum(AVLNode *cur) {
		if (!cur) return nullptr;
		while (cur->left)
		{
			cur = cur->left;
		}
		return cur;
	}
	AVLNode *search(AVLNode *cur,int value) {
		if (!cur) return nullptr;
		if (cur->data == value) return cur;
		else if (value > cur->data) return search(cur->right, value);
		else search(cur->left, value);
	}
	void destroy(AVLNode *cur) {
		if (cur)
		{
			destroy(cur->left);
			destroy(cur->right);
			delete cur;
			cur = nullptr;
		}
	}
	void PrintBinTree(AVLNode* BT)
	{
		if (BT != NULL) //树为空时结束递归
		{
			cout << BT->data;
			if (BT->left != NULL || BT->right != NULL)
			{
				cout << '(';
				if (BT->left != NULL)
				{
					PrintBinTree(BT->left);
				}
				cout << ',';
				if (BT->right != NULL)
				{
					PrintBinTree(BT->right);
				}
				cout << ')';
			}
		}
	}
	AVLNode *root;
};

int main() {
	AVLTree t;
	for (int i=0;i<10;i++)
	{
		t.Insert(i);
	}
	cout << "树高：" << t.Height() << endl;
	t.PrintBinTree();
	t.Remove(5);
	cout << endl;
	t.PrintBinTree();
	cout << endl;
}