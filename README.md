## Policy 1: Tạo tài khoản cho các nhân viên trong bảng nhân viên

### Tạo các user ứng với mã nhân viên, trong đó:

>`User 1412239 là Giám Đốc`

>`Những User :  1412193, 1412176, 1412195, ... (có rất nhiều Trưởng chi nhánh nhưng tụi em liệt kê 1 số ra) là các Trưởng chi nhánh`

>`Những User :  1412200 (P.NS), 1412001(P.KT), 1412002(P.KH), ... là các Trưởng phòng`

>`Những User :  1412173 là quản lý dự án

>`Những User : 1412186, 1412192, 1412200,1412201,1412202,1412203,1412204,1412205, ... , 1412213 là các Nhân viên bình thường`

+ #### Ví dụ từ mã nguồn:

>`create user NV401 identified by 123;`

## Policy2: Tạo role cho các vị trí phù hợp của công ty

### Tạo ra 5 role gồm: GiamDoc, Truong_CN_CTY, Truong_Phong_CTY, Truong_DA_CTY, NV_BT_CTY

+ #### Ví dụ từ mã nguồn:

>`create role giamdoc;`

>`create role truongphong;`

>`create role truongchinhanh;`

>`create role truongduan;`

>`create role nhanvien;`

### Sau đó gán từng User vừa tạo ở policy 1 cho vào từng role tương ứng.

+ #### Ví dụ từ mã nguồn:

>`GRANT GIAMDOC to “1412239”;`

## Policy 3: Chỉ trưởng phòng được phép cập nhật và thêm thông tin vào dự án (DAC) 

### Giải pháp: Gán quyền update và insert cho role TRUONGPHONG(đã được tạo ở trên) trên bảng DuAn

### Ta sử dụng role TRUONGPHONG đã được tạo từ policy 2, ta gán quyền Gán quyền update và insert cho role TRUONGPHONG trên bảng DuAn như sau :

+ #### Ví dụ từ mã nguồn:
*Lưu ý : việc viết thường hay hoa trong oracle không hề quan trọng (nền thầy đừng thắc mắc em đã test rồi)*

> `GRANT INSERT,UPDATE ON HR.DUAN TO TRUONGPHONG;`

## Policy 4: Giám đốc được phép xem thông tin dự án gồm (mã dự án, tên dự án, kinh phí, tên phòng chủ trì, tên chi nhánh chủ trì, tên trưởng dự án và tổng chi) (DAC).

### Giải pháp: Tạo 1 view và sau đó cấp quyền truy cập trên view đó cho role GIAMDOC 

### Tạo view có tên là *DUAN_CN*
##### cách để tạo view DUAN_CN : sử dụng các bảng DuAn , PhongBan , ChiNhanh , NhanVien , ChiTieu và group by trên các trưởng (maDA, tenDA, kinhphi, tenPhong, tenCN, hoTen) sau đó ta sum(soTien) là sẽ ra được yêu cầu đề bài 

+ #### Ví dụ từ mã nguồn: 

>`create view DUAN_CN `

>`as select maDA, tenDA, kinhphi, tenPhong, tenCN, hoTen, sum(soTien) as Tongchi`

>`from DuAn a, PhongBan b, ChiNhanh c, NhanVien d, ChiTieu e`

>`where a.phongChuTri = b.maPhong and b.chiNhanh = c.maCN  and a.TruongDA = d.manv and a.maDA = e.duAN`

>`group by maDA, tenDA, kinhphi, tenPhong, tenCN, hoTen`

### Thực hiện cấp quyền cho role GiamDoc trên view View_GiamDoc:

+ #### Ví dụ từ mã nguồn: 

>`grant select on DUAN_CN to GIAMDOC;`

## Policy 5: Chỉ trưởng phòng, trưởng chi nhánh được cấp quyền thực thi stored procedure cập nhật thông tin phòng ban của mình (DAC).

### Giải pháp: Tạo ra 1 procedure và sau đó cấp quyền thực thi cho rơle TRUONGPHONG và TRUONGCHINHANH.

#### Cấu trúc SP này ra sao : SP có chức năng kiểm tra các phòng ban của trưởng phòng và trưởng chi nhánh 

##### Cách làm : ta sẽ có 2 biến (checking vs checking1) 2 biến này có nhiệm vụ kiểm tra có phải phòng đó không theo Trưởng phòng (checking) và 1 biến cũng kiểm tra phòng đó không nhưng của trưởng chi nhánh (checling1) và sau đó ta giao lại nếu là 1 số != 0 thì ta sẽ cho phép update ta cùng có code sau để hiểu thêm 

+ #### Ví dụ từ mã nguồn (Khúc kiểm tra phòng):

>`select count(*) into checking from PHONGBAN where TRUONGPHONG = user  and MAPHONG = MP;`

>`select count(*) into checking1 from CHINHANH a, PHONGBAN b where a.TRUONGCHINHANH = user and a.MACN = b.CHINHANH and b.MAPHONG =
MP;`

+ #### Ví dụ từ mã nguồn (Khúc giao nhau):

>`IF (checking <= 0 and checking1 <= 0) THEN
		rollback;
		return;
	END IF;`


### Tạo procedure có tên là UpdatePhongBan5d, trong đó dữ liệu đầu vào sẽ là mã phòng cần cập nhật (MP), tên phòng cập nhật (TP), số lượng nhân viên mà mình mong muốn cập nhật (SNV) và ngày nhận chức (NNC). 

>`CREATE OR REPLACE Procedure UpdatePhongBan5d`

### Cấp quyền thực thi:

+ #### Ví dụ từ mã nguồn:

>`GRANT EXECUTE ON UpdatePhongBan5d TO TRUONGPHONG;`

>`GRANT EXECUTE ON UpdatePhongBan5d TO TRUONGCHINHANH;`

## Policy 6: Tất cả nhân viên bình thường (trừ trưởng phòng, trưởng chi nhánh và các trưởng dự án) chỉ được phép xem thông tin nhân viên trong phòng của mình, chỉ được xem lương của bản thân (VPD).

### Giải pháp: Tạo ra 2 function (nhanvien_vpd1, nhanvien_vpd2) tương ứng cho 2 policy (nhanvien_vpd1s, nhanvien_vpd2s)

+ #### Function nhanvien_vpd1 : Trả về vị từ là user là nhân viên bình thường, nếu user là trưởng phòng hoặc là trưởng chi nhánh hoặc là trưởng đề án thì được xem hết dữ liệu (ý nghĩa của function 1 là : trả về những thằng cùng phòng ban của 1 nhân viên bình thường)

+ ### Ví dụ từ mã nguồn (Cách kiểm tra và trả về vị từ cho nhân viên bình thường): 

>`select count(*) into usersx from PHONGBAN where TRUONGPHONG = user;`

>`if (usersx > 0) then
		return '';
	end if;`

>`select count(*) into usersx from CHINHANH where TRUONGCHINHANH = user;`

>`if (usersx > 0) then
		return '';
	end if;`

>`select count(*) into usersx from DUAN where TRUONGDA = user;`

>`if (usersx > 0) then
		return '';
	end if;`
  
*Nếu trải qua hết thì thực hiện trả về như sau : *

>`select maPhog into phongban from NHANVIEN where manv = user;`

>`return 'maPhog = ' || q'[']'|| phongban || q'[']';`

+ #### Function nhanvien_vpd2 : Trả về 1 vị từ chứng thực rằng đó có phải là cần được hiển thị hay chỉ là 1 thằng ko cho phép hiển thị nếu nó là 1 thằng được phép hiển thị nó sẽ trả về TRUE còn không thì sẽ trả về FALSE (nó cũng sẽ kiểm tra 1 số lệnh như function 1 để không tránh nhầm lẫn)

>`return 'MANV = ' || user;`

## Policy 7: Trưởng dự án chỉ được phép đọc, ghi thông tin chi tiêu của dự án mình quản lý (VPD).

### Giải pháp: Tạo ra vpd_chitieu_duan1 : kết với bảng DUAN để có thể lấy ra các thông tin của trương chi nhánh đó nếu kiểm tra hoàn tất thì có thể xem nếu là trưởng dự án ta sẽ chỉ trả về các chi tiêu

+ ### Ví dụ từ mã nguồn (): 

>`select count(*) into usersx from PHONGBAN where TRUONGPHONG = user;`

>`if (usersx > 0) then
		return '';
	end if;`

>`select count(*) into usersx from CHINHANH where TRUONGCHINHANH = user;`

>`if (usersx > 0) then
		return '';
	end if;`
  
*Nếu trải qua hết thì thực hiện trả về như sau : *

>`return 'EXISTS(select * from DUAN where MADA = DUAN and TRUONGDA = ' || user || ')';`

## Policy 8: Trưởng phòng chỉ được phép đọc thông tin chi tiêu của dự án trong phòng ban mình quản lý. Với những dự án không thuộc phòng ban của mình, các trưởng phòng được phép xem thông tin chi tiêu nhưng không được phép xem số tiền cụ thể (VPD).

### Giải pháp: Tạo ra vpd_chitieu_hidden_sotien1 : mục tiêu khi đã kiểm tra 1 loạt các vấn đề ở trên xem nó có phải là user đúng như yêu cầu hay không thì ta kết hợp với PHONGBAN, để lấy mã phòng ban của thằng mình đó , và dự án xem xem PB nó đang có những dự án nào 

+ ### Ví dụ từ mã nguồn (Kiểm tra nếu nó là trưởng phòng hay ko ): 
>`select count(*) into usersx from PHONGBAN where TRUONGPHONG = user;`

>`if (usersx > 0) then
		return 'EXISTS(select * from PHONGBAN , DUAN where DUAN = MADA and MAPHONG = PHONGCHUTRI and TRUONGPHONG = ' || user || ')';
	end if; return '';`
  
##### ý nghĩa của câu return trên : trả về những chi tiêu mà phòng ban mà thằng đó quản lý cũng như là check TRUE và FALSE cho từng dòng dữ liệu sau đây là gán policy vô cùng quan trọng  (bắt buộc phải có sec_relevant_cols_opt   => DBMS_RLS.ALL_ROWS để hiện thị ra những thông tin được đánh FALSE nhưng những thông tin này sẽ được ẩn đi cột lương)

>`BEGIN
  SYS.DBMS_RLS.ADD_POLICY(
  	object_schema   => 'hr',
  	object_name     => 'CHITIEU',
  	policy_name     => 'chitieu_vpd2s',
  	function_schema => 'hr',
  	policy_function => 'vpd_chitieu_hidden_sotien1',
	sec_relevant_cols       => 'SOTIEN',
 	sec_relevant_cols_opt   => DBMS_RLS.ALL_ROWS);
 END;`

## Policy 9: Mỗi dự án trong công ty có các mức độ nhạy cảm được đánh dấu bao gồm “Thông thường”, “Giới hạn”, “Bí mật”, “Bí mật cao”. Mỗi dự án có thể thuộc quyền quản lýcủa tổng công ty hoặc của 1 trong 3 chi nhánh “Tp.Hồ Chí Minh”, “Hà Nội”, “Đà Nẵng”. Mỗi dự án có thể liên quan đến các phòng ban: “Nhân sự”, “Kế toán”, “Kế hoạch”. Trưởng chi nhánh được phép truy xuất tất cả dữ liệu chi tiêu của dự án của tất cả các phòng ban thuộc quyền quản lý của mình. Trưởng chi nhánh Hà Nội được phép truy xuất dữ liệu của chi nhánh Hà Nội và tất cả các chi nhánh khác. Trưởng phòng được phép đọc dữ liệu dự án của tất cả phòng ban nhưng chỉ được phép ghi dữ liệu dự án thuộc phòng của mình. Nhân viên chỉ được đọc dữ liệu dự mình tham gia (OLS).
### 9.1 Tạo các thành phần policy.

#### Tạo policy container để chứa các thông tin về level, compartment, group, label...
>`SA_SYSDBA.CREATE_POLICY(
  policy_name => 'chitieu_policy',
  column_name => 'chitieu_label',
  default_options => 'no_control'
);`

#### Tạo các level “Thông thường”, “Giới hạn”, “Bí mật”, “Bí mật cao”.
+ ### Ví dụ từ mã nguồn:
>`SA_COMPONENTS.CREATE_LEVEL('duan_policy',10,'BT','Binh_thuong');`

#### Tạo các compartment “Nhân sự”, “Kế toán”, “Kế hoạch”.
+ ### Ví dụ từ mã nguồn:
>`SA_COMPONENTS.CREATE_COMPARTMENT('duan_policy',10,'NS','Nhan_su');`

#### Tạo các group "TPHCM", "HN", "DN"
+ ### Ví dụ từ mã nguồn:
>`SA_COMPONENTS.CREATE_GROUP('duan_policy',10,'TPHCM','TP_Ho_Chi_Minh');`

#### Dữ liệu có nhãn không có group thì là thuộc chi nhánh tổng công ty (không phụ thuộc vào chi nhánh)

#### Tạo ra các label tương ứng
+ ### Ví dụ từ mã nguồn:
>`SA_LABEL_ADMIN.CREATE_LABEL('duan_policy',100,'BT:NS:TPHCM');
SA_LABEL_ADMIN.CREATE_LABEL('duan_policy',101,'GH:NS:TPHCM');
SA_LABEL_ADMIN.CREATE_LABEL('duan_policy',102,'BM:NS:TPHCM');
SA_LABEL_ADMIN.CREATE_LABEL('duan_policy',133,'GH:KH:DN');
SA_LABEL_ADMIN.CREATE_LABEL('duan_policy',134,'BM:KH:DN');
SA_LABEL_ADMIN.CREATE_LABEL('duan_policy',135,'BMC:KH:DN');`

### 9.2 Trưởng chi nhánh được phép truy xuất tất cả dữ liệu chi tiêu của dự án của tất cả các phòng ban thuộc quyền quản lý của mình.
##### Sau đó gán nhãn cho các user là Trưởng chi nhánh
+ ### Ví dụ từ mã nguồn:
>`--Gan nhan co truong chi nhanh TPHCM 1412193 (CN001)
BEGIN
 SA_USER_ADMIN.SET_LEVELS (
  policy_name   => 'DUAN_POLICY',
  user_name     => '1412193', 
  max_level     => 'BMC',
  min_level     => 'BT',
  def_level     => 'BMC',
  row_level     => 'BMC');
 SA_USER_ADMIN.SET_COMPARTMENTS (
  policy_name   => 'DUAN_POLICY',
  user_name     => '1412193', 
  read_comps    => 'NS,KT,KH',
  write_comps   => '',
  def_comps     => 'NS,KT,KH',
  row_comps     => '');
 SA_USER_ADMIN.SET_GROUPS (
  policy_name   => 'DUAN_POLICY',
  user_name     => '1412193', 
  read_groups   => 'TPHCM',
  write_groups  => '',
  def_groups    => 'TPHCM',
  row_groups    => '');
END;
/`

### 9.3 Trưởng chi nhánh Hà Nội được phép truy xuất dữ liệu của chi nhánh Hà Nội và tất cả các chi nhánh khác.
>`--Gan nhan cho truong chi nhanh HN 1412195 (CN003)
BEGIN
 SA_USER_ADMIN.SET_LEVELS (
  policy_name   => 'DUAN_POLICY',
  user_name     => '1412195', 
  max_level     => 'BMC',
  min_level     => 'BT',
  def_level     => 'BMC',
  row_level     => 'BMC');
 SA_USER_ADMIN.SET_COMPARTMENTS (
  policy_name   => 'DUAN_POLICY',
  user_name     => '1412195', 
  read_comps    => 'NS,KT,KH',
  write_comps   => NULL,
  def_comps     => 'NS,KT,KH',
  row_comps     => NULL);
 SA_USER_ADMIN.SET_GROUPS (
  policy_name   => 'DUAN_POLICY',
  user_name     => '1412195', 
  read_groups   => 'DN,HN,TPHCM',
  write_groups  => NULL,
  def_groups    => 'DN,HN,TPHCM',
  row_groups    => NULL);
END;
/`

### 9.4 Trưởng phòng được phép đọc dữ liệu dự án của tất cả phòng ban nhưng chỉ được phép ghi dữ liệu dự án thuộc phòng của mình.
+ ### Ví dụ từ mã nguồn:
>`--Gan nhan cho truong phong PB001 NS (phong Nhan su) 1412200
BEGIN
 SA_USER_ADMIN.SET_LEVELS (
  policy_name   => 'DUAN_POLICY',
  user_name     => '1412200', 
  max_level     => 'BMC',
  min_level     => 'BT',
  def_level     => 'BMC',
  row_level     => 'BMC');
 SA_USER_ADMIN.SET_COMPARTMENTS (
  policy_name   => 'DUAN_POLICY',
  user_name     => '1412200', 
  read_comps    => 'NS,KT,KH',
  write_comps   => 'NS',
  def_comps     => 'NS,KT,KH',
  row_comps     => 'NS');
 SA_USER_ADMIN.SET_GROUPS (
  policy_name   => 'DUAN_POLICY',
  user_name     => '1412200', 
  read_groups   => 'TPHCM', --TPHCM vi 1412200 la chi nhanh TPHCM
  write_groups  => 'TPHCM',
  def_groups    => 'TPHCM',
  row_groups    => 'TPHCM');
END;
/`

#### Gán nhãn cho dữ liệu bảng DuAn
+ ### Ví dụ từ mã nguồn:
>`UPDATE hr.duan SET duan_label = CHAR_TO_LABEL('DUAN_POLICY','BT:NS:TPHCM') where mada='DA001';`

### 9.5 Nhân viên chỉ được đọc dữ liệu dự mình tham gia.
+ ### Ví dụ từ mã nguồn:
>`SA_USER_ADMIN.SET_USER_LABELS('duan_policy','1412210','BT:NS:TPHCM');`

## Policy 10: Mỗi thông tin thu chi sẽ được đánh dấu các mức độ “Nhạy cảm”, “Không nhạy cảm”, “Bí mật” và thuộc các nhóm như “Lương”, “Quản lý”, “Vật liệu”. Nhân viên phụ trách đủ các lĩnh vực, có cấp độ phù hợp mới được phép truy xuất dữ liệu thu chi. Ngoài ra, mỗi thông tin thu chi còn quy định cấp “Quản lý” hay “Nhân viên” để xác định dữ liệu này thuộc cấp quản lý của nhân viên hay quản lý dự án. Quản lý có thể xem tất cả thông tin thu chi của nhân viên (OLS).

#### Tạo các level “Nhạy cảm”, “Không nhạy cảm”, “Bí mật”.
+ ### Ví dụ từ mã nguồn: 
>`SA_COMPONENTS.CREATE_LEVEL('chitieu_policy',10,'KNC','Nhay_cam');`

#### Tạo các compartment “Lương”, “Quản lý”, “Vật liệu”.
+ ### Ví dụ từ mã nguồn: 
>`SA_COMPONENTS.CREATE_COMPARTMENT('chitieu_policy',10,'L','Luong');`

#### Tạo các group "Nhân viên", "Quản lý"
>`SA_COMPONENTS.CREATE_GROUP('chitieu_policy',10,'QL','Quan_ly');`

#### Tạo ra các label tương ứng
+ ### Ví dụ từ mã nguồn:
>`SA_LABEL_ADMIN.CREATE_LABEL('chitieu_policy',1000,'KNC:L:QL');
  SA_LABEL_ADMIN.CREATE_LABEL('chitieu_policy',1001,'NC:L:QL');
  SA_LABEL_ADMIN.CREATE_LABEL('chitieu_policy',1016,'NC:VL:NV');
  SA_LABEL_ADMIN.CREATE_LABEL('chitieu_policy',1017,'BM:VL:NV');`
  
### 10.1 Nhân viên phụ trách đủ các lĩnh vực, có cấp độ phù hợp mới được phép truy xuất dữ liệu thu chi.
+ ### Ví dụ từ mã nguồn:
>`EXECUTE SA_USER_ADMIN.SET_USER_LABELS('CHITIEU_POLICY','1412193','NC:L,QL:NV');
EXECUTE SA_USER_ADMIN.SET_USER_LABELS('CHITIEU_POLICY','1412257','BM:VL:NV');`

### 10.2 Ngoài ra, mỗi thông tin thu chi còn quy định cấp “Quản lý” hay “Nhân viên” để xác định dữ liệu này thuộc cấp quản lý của nhân viên hay quản lý dự án. Quản lý có thể xem tất cả thông tin thu chi của nhân viên.
#### Dữ liệu thuộc group "QL" hoặc "NV" có thể phân biệt được cấp quản lý của nhân viên hoặc quản lý dự án

#### Group "Nhân viên" nhận group "Quản lý" làm cha nên user có nhãn "Quản lý" có thể truy xuất được cả dữ liệu nhãn "Nhân viên"
>`SA_COMPONENTS.CREATE_GROUP('chitieu_policy',20,'NV','Nhan_vien', 'QL');`
